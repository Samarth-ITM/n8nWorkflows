Improve AI Agent System Prompts with GPT-4o Feedback Analysis and Email Delivery

https://n8nworkflows.xyz/workflows/improve-ai-agent-system-prompts-with-gpt-4o-feedback-analysis-and-email-delivery-4197


# Improve AI Agent System Prompts with GPT-4o Feedback Analysis and Email Delivery

### 1. Workflow Overview

This workflow, named **System Prompt Tuner**, is designed to improve autonomous AI agents' system prompts by collecting user feedback on the agent’s current behavior and performance, analyzing this feedback via GPT-4o, and delivering a refined system prompt via email. It targets AI developers or prompt engineers seeking to iteratively tune AI agent prompts to enhance their accuracy and effectiveness.

The workflow is logically structured into the following blocks:

- **1.1 Input Reception:** Captures detailed user input about the AI agent’s identity, purpose, current prompt, and performance feedback.
- **1.2 AI Analysis and Improvement Generation:** Uses a LangChain AI Agent powered by GPT-4o and a structured output parser to analyze the input and generate an improved system prompt in JSON format.
- **1.3 Email Notification:** Sends a formatted HTML email with the improved prompt, original prompt, and a summary of improvements to a predefined email address.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Collects comprehensive user input describing the AI agent’s current operational context, including qualitative feedback on what works and what doesn’t, plus optional example inputs and outputs.

**Nodes Involved:**  
- User inputs

**Node Details:**  

- **User inputs**  
  - Type: Form Trigger  
  - Purpose: Starts the workflow when a user submits a form  
  - Configuration:  
    - Form title: "Agent Prompt Tuner"  
    - Button label: "Send For Evaluation"  
    - Fields:  
      - Agent Name (required text)  
      - Agent Purpose (required multiline textarea)  
      - What's Working? (required multiline textarea)  
      - What's Not Working? (required multiline textarea)  
      - Current System Prompt (required multiline textarea)  
      - Sample Input (Prompt) (optional multiline textarea)  
      - Example Output (optional multiline textarea)  
    - Description: "Enhances autonomous agent system prompts based on user provided descriptions of behavior and examples"  
  - Inputs: Webhook trigger from form submission  
  - Outputs: JSON object with all form field values  
  - Edge Cases/Potential Failures:  
    - Missing required fields will block submission  
    - Malformed inputs (e.g., very large text) could cause delays or truncation issues  
    - User abandonment before submission means no workflow execution

---

#### 2.2 AI Analysis and Improvement Generation

**Overview:**  
Receives the user inputs, forwards them to a LangChain AI Agent configured with GPT-4o, then parses the agent’s JSON-formatted response into structured output.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**  

- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Core logical processor that formats and sends the user input to the GPT-4o model and receives improvement suggestions  
  - Configuration:  
    - Input text template includes all user fields formatted as descriptive instructions for analysis  
    - System message sets the role as an expert AI prompt engineer, instructing it to produce a JSON with two fields: `updated_system_prompt` and `summary_of_improvements`  
    - Output parser enabled to ensure strict JSON output  
  - Inputs: JSON from "User inputs" node  
  - Outputs: JSON with improved system prompt and summary  
  - Connected to: OpenAI Chat Model (language model), Structured Output Parser (to parse response)  
  - Edge Cases/Potential Failures:  
    - API authentication errors or rate limits from OpenAI  
    - Malformed user input causing model confusion or output errors  
    - Failure to produce valid JSON (partially mitigated by structured output parser)  
    - Timeout or latency issues from OpenAI API

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Node  
  - Role: Provides GPT-4o model interface for the AI Agent  
  - Configuration: Uses GPT-4o model with default options, authenticated via OpenAI API credentials  
  - Inputs: Requests from AI Agent  
  - Outputs: Raw model response to AI Agent  
  - Edge Cases: API quota exceeded, network errors, version compatibility

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI Agent’s JSON string output into structured JSON object for downstream use  
  - Configuration: JSON schema example enforces two fields: `updated_system_prompt` and `summary_of_improvements`  
  - Inputs: AI Agent output  
  - Outputs: JSON object with parsed fields  
  - Edge Cases: Parsing failure if model output does not conform to JSON schema

---

#### 2.3 Email Notification

**Overview:**  
Formats an HTML email incorporating the improved prompt, the original prompt, and the summary of improvements, then sends it to a configured recipient via Gmail OAuth2.

**Nodes Involved:**  
- Gmail

**Node Details:**  

- **Gmail**  
  - Type: Gmail Node (Send Email)  
  - Role: Sends notification email with prompt improvement details  
  - Configuration:  
    - Recipient email hardcoded as "user@example.com" (should be customized)  
    - Subject dynamically includes the Agent Name from user input  
    - Message body is a detailed HTML email with styled sections for improved prompt, original prompt, and summary of improvements, referencing relevant data via expressions from previous nodes  
    - Uses Gmail OAuth2 credentials for authentication  
  - Inputs: Improved prompt JSON from AI Agent (via connections) and user inputs  
  - Outputs: Email sent confirmation output  
  - Edge Cases/Potential Failures:  
    - Authentication or credential issues with Gmail OAuth2  
    - Email quota or sending limits reached  
    - HTML rendering issues on some email clients  
    - Recipient email hardcoded and may need dynamic configuration for broader use

---

### 3. Summary Table

| Node Name            | Node Type                          | Functional Role                        | Input Node(s)       | Output Node(s)   | Sticky Note                                                                                      |
|----------------------|----------------------------------|-------------------------------------|---------------------|------------------|------------------------------------------------------------------------------------------------|
| User inputs          | Form Trigger                     | Captures user feedback for tuning  | (Webhook trigger)    | AI Agent         | "Enhances autonomous agent system prompts based on user provided descriptions of behavior."    |
| OpenAI Chat Model    | LangChain OpenAI Chat Model      | Provides GPT-4o model for AI Agent  | AI Agent (ai_languageModel) | AI Agent    |                                                                                                |
| Structured Output Parser | LangChain Structured Output Parser | Parses GPT output into JSON         | AI Agent (ai_outputParser) | AI Agent        |                                                                                                |
| AI Agent             | LangChain Agent                  | Analyzes feedback and generates improved prompt | User inputs          | Gmail            |                                                                                                |
| Gmail                | Gmail Node (Send Email)           | Sends email with prompt improvements | AI Agent             | -                | Email formatted in HTML with improved prompt, original prompt, and summary of improvements.    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "User inputs" node:**  
   - Type: Form Trigger  
   - Configure webhook and form with fields:  
     - Agent Name (required text)  
     - Agent Purpose (required textarea)  
     - What's Working? (required textarea)  
     - What's Not Working? (required textarea)  
     - Current System Prompt (required textarea)  
     - Sample Input (Prompt) (optional textarea)  
     - Example Output (optional textarea)  
   - Set form title to "Agent Prompt Tuner" and button label to "Send For Evaluation"  
   - Provide form description: "Enhances autonomous agent system prompts based on user provided descriptions of behavior and examples"  

2. **Create the "OpenAI Chat Model" node:**  
   - Type: LangChain OpenAI Chat Model  
   - Select model: GPT-4o  
   - Configure OpenAI API credentials with valid key  
   - Leave options default  

3. **Create the "Structured Output Parser" node:**  
   - Type: LangChain Structured Output Parser  
   - Define JSON schema example with fields:  
     ```json
     {
       "updated_system_prompt": "string",
       "summary_of_improvements": "string"
     }
     ```  
   - This enforces parsing of the AI Agent output  

4. **Create the "AI Agent" node:**  
   - Type: LangChain Agent  
   - Set `text` parameter with an expression combining all form fields to form a detailed input text for analysis (see node details section for template)  
   - Configure system message to instruct the agent as an expert prompt engineer to analyze input and return a JSON with `updated_system_prompt` and `summary_of_improvements` only  
   - Enable output parser and select the "Structured Output Parser" node  
   - Connect the "OpenAI Chat Model" node as the language model source  
   - Connect input from "User inputs" node  

5. **Create the "Gmail" node:**  
   - Type: Gmail (Send Email)  
   - Configure Gmail OAuth2 credentials with appropriate permissions  
   - Set recipient email (default is "user@example.com", customize as needed)  
   - Configure subject line with expression: `Improved prompt for: {{ $('User inputs').item.json['Agent Name'] }}`  
   - Compose HTML email body using expression referencing:  
     - Improved prompt: `{{ $json.output.updated_system_prompt }}`  
     - Original prompt: `{{ $('User inputs').item.json['Current System Prompt'] }}`  
     - Summary: `{{ $json.output.summary_of_improvements }}`  
   - Connect input from "AI Agent" node  

6. **Connect nodes in sequence:**  
   - "User inputs" → "AI Agent"  
   - "OpenAI Chat Model" → "AI Agent" (as language model)  
   - "Structured Output Parser" → "AI Agent" (as output parser)  
   - "AI Agent" → "Gmail"  

7. **Activate and test the workflow:**  
   - Deploy the webhook URL for the form trigger  
   - Submit test inputs through the form  
   - Verify AI-generated prompt improvements are emailed correctly  

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                     |
|-------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The AI Agent’s system prompt enforces strict JSON output to ensure reliable parsing and integration. | Important for preventing parsing errors and downstream processing failures.                        |
| The Gmail recipient is currently fixed; consider parameterizing for multi-user environments.      | Allows scaling the workflow for multiple users or dynamic routing.                                |
| Uses GPT-4o model, which may require appropriate OpenAI subscription or quota management.         | Check OpenAI account plan and usage limits before production deployment.                           |
| Styling in the email uses inline CSS optimized for modern email clients.                          | Ensures readable and professional formatting across platforms.                                   |
| Workflow respects n8n version 2.x node versions and LangChain integration requirements.           | Verify node compatibility when importing or upgrading n8n instances.                             |

---

**Disclaimer:** The text provided is exclusively derived from an n8n automated workflow. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.