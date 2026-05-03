Automated Email Inquiry Processing & Routing with Gmail and Gemini AI

https://n8nworkflows.xyz/workflows/automated-email-inquiry-processing---routing-with-gmail-and-gemini-ai-6633


# Automated Email Inquiry Processing & Routing with Gmail and Gemini AI

### 1. Workflow Overview

This workflow automates the processing and routing of inquiry emails received via Gmail, leveraging AI to understand customer intent and respond accordingly. It is designed for businesses across any industry to streamline handling of customer inquiries, distinguishing between simple information requests and booking actions.

**Logical Blocks:**

- **1.1 Input Reception:** Gmail Trigger node listens for new incoming emails from a specified sender.
- **1.2 AI Processing:** The AI Agent node analyzes the email content, classifies the intent, extracts relevant data, and generates a professional response.
- **1.3 JSON Parsing:** Code node cleans and parses the AI Agent’s raw JSON output for structured use.
- **1.4 Conditional Routing:** If node routes the workflow based on the AI-detected action (availability check or booking).
- **1.5 Email Response & Forwarding:** Gmail nodes send replies to customers or forward booking requests internally.
- **1.6 Support Nodes:** Includes a Wait node for synchronization and a disabled HTTP Request node configured for potential external search integration.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for new emails from a specific Gmail account and triggers the workflow.

- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**  
  - **Gmail Trigger**  
    - Type: Trigger node for Gmail  
    - Configuration: Monitors inbox for new emails from sender "acdert@gmail.com", polling every few minutes.  
    - Expressions: Uses filter for sender email.  
    - Inputs: None (trigger)  
    - Outputs: Email data JSON passed forward.  
    - Version: 1.2  
    - Edge Cases: Gmail OAuth2 authentication errors, email fetch delays, filter mismatch (emails from other senders ignored).  
    - Sub-workflow: None

---

#### 2.2 AI Processing

- **Overview:**  
  Uses an AI agent powered by Google Gemini to analyze incoming email content, determine customer intent, extract data, and draft replies.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model (linked as language model)  
  - HTTP Request (disabled)

- **Node Details:**  
  - **AI Agent**  
    - Type: Langchain Agent node integrating AI language model  
    - Configuration:  
      - Input text bound to incoming email content (`={{ $json.text }}`).  
      - System message instructs AI to classify intent as "check_info" or "forward_action", extract relevant structured data, and generate professional replies with behavioral logic (70% availability assumed, alternatives suggested if unavailable, warm tone).  
      - Prompt type: Define (custom prompt).  
    - Inputs: Email text from Gmail Trigger  
    - Outputs: Raw AI JSON response string  
    - Version: 1.9  
    - Edge Cases: AI model failures, incomplete or ambiguous email text, unexpected AI output formatting, API rate limits.  
    - Sub-workflow: Uses Google Gemini Chat Model as language model backend.

  - **Google Gemini Chat Model**  
    - Type: AI language model node  
    - Configuration: Uses "models/gemini-2.5-pro" with default options.  
    - Inputs: AI Agent calls this node for text generation.  
    - Outputs: AI-generated text for AI Agent.  
    - Version: 1  
    - Edge Cases: Service unavailability, authentication issues, API quota exceeded.  
    - Note: Node currently has execution issues, fallback or alternative AI may be needed.

  - **HTTP Request (Disabled)**  
    - Type: HTTP Request tool node  
    - Configuration: POST request to Serper search API with API key; currently disabled.  
    - Inputs/Outputs: Not active in current workflow.  
    - Edge Cases: Network errors, API errors, rate limiting if enabled.

---

#### 2.3 JSON Parsing

- **Overview:**  
  Parses the AI Agent’s raw string output to extract clean JSON data for downstream logic.

- **Nodes Involved:**  
  - Code

- **Node Details:**  
  - **Code**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Iterates over input items, removes markdown fences (``````), extracts substring between first `{` and last `}`, attempts JSON parse.  
      - On parse error, outputs error message and raw string for debugging.  
    - Inputs: Raw AI Agent output  
    - Outputs: Parsed JSON object with action, reasoning, extracted data, reply, etc.  
    - Version: 2  
    - Edge Cases: Malformed AI output, missing or extra characters causing parse failure.

---

#### 2.4 Conditional Routing

- **Overview:**  
  Decides workflow path based on AI-detected intent to send either an availability response or forward booking details.

- **Nodes Involved:**  
  - Wait for Data  
  - If

- **Node Details:**  
  - **Wait for Data**  
    - Type: Wait node  
    - Configuration: Default (no specific wait time set)  
    - Purpose: Synchronization point between Code parsing and conditional check.  
    - Inputs: Parsed JSON from Code node  
    - Outputs: Decisions to If node  
    - Edge Cases: Potential unnecessary delay if no wait parameters specified.

  - **If**  
    - Type: Conditional node  
    - Configuration: Checks if `action` field equals `"check_availability"`.  
    - Inputs: JSON from Wait node  
    - Outputs:  
      - True branch: availability check email node  
      - False branch: Forward booking email node  
    - Version: 2.2  
    - Edge Cases: Missing or unexpected `action` values, case sensitivity issues.

---

#### 2.5 Email Response & Forwarding

- **Overview:**  
  Sends appropriate emails based on intent: replies to customer inquiries or forwards booking requests internally.

- **Nodes Involved:**  
  - availability check email (Gmail)  
  - Forward booking email (Gmail)

- **Node Details:**  
  - **availability check email**  
    - Type: Gmail node (send email)  
    - Configuration:  
      - Sends to customer email extracted from AI data.  
      - Email body is generated content from AI (via Code node’s `email_response`).  
      - Subject: "Inquiry Details"  
      - Email type: Plain text  
    - Inputs: True branch from If node  
    - Credentials: Gmail OAuth2 (same as trigger)  
    - Version: 2.1  
    - Edge Cases: Email sending errors, missing customer email, formatting issues.

  - **Forward booking email**  
    - Type: Gmail node (send email)  
    - Configuration:  
      - Sends to internal email address `abc@gmail.com`.  
      - Email body includes customer data fields (name, email, phone, original message), internal notes, and booking summary from AI output.  
      - Subject: "Inquiry Details"  
      - Email type: Plain text  
    - Inputs: False branch from If node  
    - Credentials: Gmail OAuth2  
    - Version: 2.1  
    - Edge Cases: Missing internal email address configuration, data extraction errors.

---

#### 2.6 Support & Documentation Nodes

- **Overview:**  
  Sticky Notes provide documentation and explanations within the workflow editor.

- **Nodes Involved:**  
  - Workflow Overview (sticky note)  
  - AI Agent and Code Explanation (sticky note)  
  - Conditional Routing and Email Responses (sticky note)

- **Node Details:**  
  - Sticky notes contain detailed explanations of workflow logic, AI role, node purpose, and routing decisions, improving maintainability and collaboration.

---

### 3. Summary Table

| Node Name                  | Node Type                    | Functional Role                             | Input Node(s)         | Output Node(s)               | Sticky Note                                                                                      |
|----------------------------|------------------------------|---------------------------------------------|-----------------------|-----------------------------|-------------------------------------------------------------------------------------------------|
| Gmail Trigger              | Gmail Trigger (trigger)       | Listens for new incoming emails             | None                  | AI Agent                    | Workflow Overview 📧 - Describes overall workflow process                                        |
| AI Agent                  | Langchain Agent               | AI analysis of email content and intent     | Gmail Trigger         | Code                        | AI Agent and Code Node Explained 🤖 - Details AI processing and code parsing                    |
| Google Gemini Chat Model   | AI Language Model             | Backend AI model for AI Agent                | AI Agent (as LM)      | AI Agent                    | AI Agent and Code Node Explained 🤖 - Notes execution issues with Gemini model                  |
| HTTP Request              | HTTP Request (disabled)       | Disabled external search integration         | None                  | AI Agent (ai_tool)          | AI Agent and Code Node Explained 🤖 - Disabled HTTP node setup for Serper search                |
| Code                      | Code                         | Parses AI raw JSON output                     | AI Agent              | Wait for Data               | AI Agent and Code Node Explained 🤖 - Cleans and parses AI output JSON                          |
| Wait for Data             | Wait                         | Synchronization before decision making       | Code                  | If                          | Conditional Routing & Email Responses 🚦 - Part of routing logic                               |
| If                        | If                           | Routes workflow based on AI action field     | Wait for Data         | availability check email, Forward booking email | Conditional Routing & Email Responses 🚦 - Decision node for intent routing                      |
| availability check email  | Gmail                        | Sends availability response to customer     | If (true branch)      | None                       | Conditional Routing & Email Responses 🚦 - Sends customer reply                                |
| Forward booking email     | Gmail                        | Forwards booking details internally          | If (false branch)     | None                       | Conditional Routing & Email Responses 🚦 - Forwards booking info to internal email             |
| Workflow Overview         | Sticky Note                  | Contains high-level workflow explanation     | None                  | None                       | Workflow Overview 📧 - Overall workflow summary                                               |
| AI Agent and Code Explanation | Sticky Note                  | Explains AI Agent and Code node functionality| None                  | None                       | AI Agent and Code Node Explained 🤖 - Detailed AI processing explanation                      |
| Conditional Routing and Email Responses | Sticky Note                  | Explains decision and email sending logic    | None                  | None                       | Conditional Routing & Email Responses 🚦 - Describes routing and email response logic          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node:**  
   - Type: Gmail Trigger  
   - Set filter: sender equals "acdert@gmail.com"  
   - Poll interval: every few minutes (default)  
   - Connect Gmail OAuth2 credentials with proper scopes.

2. **Add AI Agent node:**  
   - Type: Langchain Agent  
   - Input text: bind to `{{$json.text}}` from Gmail Trigger  
   - Configure system message prompt:  
     - Explains classification of intent (`check_info` or `forward_action`)  
     - Instructs to extract structured data and generate professional reply  
     - Includes behavior logic for availability and tone  
   - Set prompt type to "define"  
   - Connect Google Gemini Chat Model as AI language model node  
   - Ensure Google Gemini API credentials (Google PaLM API key) are configured.

3. **Add Google Gemini Chat Model node:**  
   - Type: AI language model  
   - Model name: "models/gemini-2.5-pro"  
   - Use Google Palm API credentials  
   - Connect as AI language model for AI Agent node.

4. **(Optional) Add HTTP Request node (disabled):**  
   - Configure POST to Serper API with API key  
   - Set body parameter "q" dynamically (not currently used).

5. **Add Code node:**  
   - Type: Code (JavaScript)  
   - Paste cleaning and parsing JS code to extract valid JSON from AI Agent output string.  
   - Connect input from AI Agent output.

6. **Add Wait for Data node:**  
   - Type: Wait  
   - Leave default parameters  
   - Connect input from Code node output.

7. **Add If node:**  
   - Type: If (conditional)  
   - Condition: Check if `{{$json.action}}` equals "check_availability" (case sensitive)  
   - Connect input from Wait node output.

8. **Add availability check email node:**  
   - Type: Gmail (send email)  
   - Configure "Send To": `{{$json.customer_data.email}}`  
   - Message body: `{{$('Code').item.json.email_response}}` (parsed AI reply)  
   - Subject: "Inquiry Details"  
   - Use Gmail OAuth2 credentials  
   - Connect as True branch output of If node.

9. **Add Forward booking email node:**  
   - Type: Gmail (send email)  
   - Configure "Send To": `abc@gmail.com` (internal email)  
   - Message body includes:  
     - Customer name, email, phone, original message from parsed AI data  
     - Internal note and booking summary fields from AI output  
   - Subject: "Inquiry Details"  
   - Use Gmail OAuth2 credentials  
   - Connect as False branch output of If node.

10. **(Optional) Add Sticky Notes:**  
    - Add notes for workflow overview, AI agent explanation, and conditional routing details to document workflow steps for maintainers.

11. **Activate workflow:**  
    - Check all credentials and permissions.  
    - Test with emails from the specified sender.  
    - Monitor logs for errors in AI parsing and email sending.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                                                                                          |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow demonstrates AI-powered email intent classification and automated response generation using Google Gemini AI integrated within n8n.                                                                                            | Workflow description                                                                                   |
| Google Gemini Chat Model node currently has execution issues; consider fallback or alternative AI models if needed to maintain workflow reliability.                                                                                         | AI Agent and Code Node Explanation sticky note                                                        |
| The disabled HTTP Request node is configured for Serper API search calls, which may be enabled in future for enhanced AI context or external data lookup.                                                                                    | AI Agent and Code Node Explanation sticky note                                                        |
| Gmail OAuth2 credentials must have appropriate scopes for reading and sending emails. Ensure proper consent and security compliance for production use.                                                                                       | Gmail Trigger and Gmail nodes configuration                                                           |
| For best results, customize the AI system message prompt to match your business tone, services, and data extraction needs.                                                                                                                    | AI Agent node configuration                                                                            |
| The workflow’s design is generic and can adapt to any industry by modifying AI prompts and email templates accordingly.                                                                                                                      | Workflow Overview sticky note                                                                          |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created using n8n, an integration and automation tool. This process strictly adheres to the current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.