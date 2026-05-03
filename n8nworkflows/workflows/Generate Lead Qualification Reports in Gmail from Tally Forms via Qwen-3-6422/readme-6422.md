Generate Lead Qualification Reports in Gmail from Tally Forms via Qwen-3

https://n8nworkflows.xyz/workflows/generate-lead-qualification-reports-in-gmail-from-tally-forms-via-qwen-3-6422


# Generate Lead Qualification Reports in Gmail from Tally Forms via Qwen-3

### 1. Workflow Overview

This workflow automates the generation of lead qualification reports in Gmail based on Tally form submissions, leveraging advanced AI models hosted on OpenRouter. It is designed for sales or marketing teams in AI services agencies to streamline lead assessment and follow-up.

**Target Use Cases:**
- Automatically process lead information submitted via Tally forms.
- Generate detailed AI-powered lead qualification summaries and recommendations.
- Send these reports by email to a designated recipient.
- Facilitate rapid evaluation and prioritization of sales leads.

**Logical Blocks:**

- **1.1 Input Reception**  
  Receives and captures incoming form responses via webhook from Tally.

- **1.2 Email Setup**  
  Sets the recipient email address for the outgoing report.

- **1.3 Lead Qualification AI Processing**  
  Processes form data through multiple AI models to generate a comprehensive lead qualification report.

- **1.4 Email Dispatch**  
  Sends the AI-generated report as an email to the specified recipient.

- **1.5 Documentation and Instructions**  
  Sticky notes providing setup instructions and reminders to configure email and credentials.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures new lead data submitted via Tally form through a webhook node.

- **Nodes Involved:**  
  - Tally Form Response

- **Node Details:**  

  - **Tally Form Response**  
    - Type: Webhook (HTTP POST endpoint)  
    - Role: Receives incoming HTTP POST data from Tally form submissions.  
    - Configuration: Webhook path set to a unique identifier; listens for POST requests only.  
    - Key Variables: Form data fields extracted from JSON payload under `body.data.fields`.  
    - Input: External HTTP POST from Tally form integration.  
    - Output: JSON data with structured form responses forwarded downstream.  
    - Edge Cases: Possible failures if Tally sends malformed data or network interruptions occur; webhook URL must be publicly accessible.  
    - No sub-workflow reference.

#### 1.2 Email Setup

- **Overview:**  
  Assigns the recipient email address for the outgoing qualification report.

- **Nodes Involved:**  
  - Set Email  
  - Sticky Note (Set your email)

- **Node Details:**  

  - **Set Email**  
    - Type: Set node  
    - Role: Defines a string variable `toEmail` containing the recipient's email address.  
    - Configuration: Hardcoded placeholder value `"insert-your-email"` must be replaced by the user.  
    - Expressions: None dynamic; static string assignment.  
    - Input: Receives data from webhook node.  
    - Output: Passes data and attached `toEmail` field downstream.  
    - Edge Cases: If not updated, emails will be sent to a placeholder address (likely failing or misdirected).  
    - No sub-workflow reference.

  - **Sticky Note (Set your email)**  
    - Type: Sticky Note  
    - Role: Instructional note reminding user to specify the recipient email in the Set Email node.  
    - No inputs/outputs.

#### 1.3 Lead Qualification AI Processing

- **Overview:**  
  Uses structured Tally form data as input into multiple AI models to generate a detailed lead qualification summary and recommendations.

- **Nodes Involved:**  
  - Qualify Lead  
  - Qwen3-07-25 (OpenRouter LLM node)  
  - Gemini 2.5 pro (OpenRouter LLM node)

- **Node Details:**  

  - **Qualify Lead**  
    - Type: Chain LLM node from Langchain integration (n8n-nodes-langchain.chainLlm)  
    - Role: Orchestrates the lead qualification prompt sent to AI models, integrates input fields, and defines detailed instructions to the model.  
    - Configuration:  
      - Text template dynamically interpolates fields from the Tally response node using expressions like `{{ $('Tally Form Response').item.json.body.data.fields[0].value }}` for each question.  
      - Prompt includes explicit instructions and security rules to prevent prompt injections and ensure professional output.  
      - Output format defines sections such as Summary, AI Challenges, Recommendations, etc.  
      - Uses two AI models in parallel: "Qwen3-07-25" and "Gemini 2.5 pro" nodes are linked as language model providers.  
    - Input: JSON from Set Email node (which receives from Tally Form Response).  
    - Output: Structured text summary from AI, passed to email sending.  
    - Edge Cases:  
      - AI model call failures (timeouts, API limits).  
      - Missing or malformed form data leading to incomplete reports.  
      - Expression errors if field indices change or data is missing.  
    - Version requirements: Compatible with n8n Langchain nodes v1.7+.  
    - Sub-workflow: None; integrates with external AI model credentials.

  - **Qwen3-07-25** and **Gemini 2.5 pro**  
    - Type: OpenRouter API language model nodes.  
    - Role: Provide AI processing engines for the Chain LLM node.  
    - Configuration: Set to specific models ("qwen/qwen3-235b-a22b-07-25" and "google/gemini-2.5-pro") connected via OpenRouter API credentials.  
    - Input: Prompt text from Chain LLM node.  
    - Output: AI-generated text response.  
    - Edge Cases: API authentication errors, rate limits, network failures.  
    - Credentials: Requires valid OpenRouter API keys.

#### 1.4 Email Dispatch

- **Overview:**  
  Sends the AI-generated lead qualification report to the configured email address.

- **Nodes Involved:**  
  - Send a message

- **Node Details:**  

  - **Send a message**  
    - Type: Gmail node (n8n-nodes-base.gmail)  
    - Role: Sends an email containing the AI-generated report text.  
    - Configuration:  
      - Recipient email dynamically set via expression `={{ $('Set Email').item.json.toEmail }}`.  
      - Email subject hardcoded as "New form submission".  
      - Message body set dynamically from AI output JSON field `$json.text`.  
      - Email type set to plain text without attribution.  
    - Credentials: Uses Gmail OAuth2 credential with required Gmail API scopes.  
    - Input: AI output text from Qualify Lead.  
    - Output: None (end node).  
    - Edge Cases:  
      - Failed authentication or expired OAuth tokens.  
      - Gmail API quota exceeded.  
      - Invalid or missing recipient email.  
    - Version: Node version 2.1.

#### 1.5 Documentation and Instructions

- **Overview:**  
  Provides user guidance for setup and prerequisites through sticky notes.

- **Nodes Involved:**  
  - Sticky Note1 (Requirements)

- **Node Details:**  

  - **Sticky Note1**  
    - Type: Sticky Note  
    - Content: Lists requirements such as Tally account setup, form creation, webhook URL configuration, email customization, and OpenRouter connection.  
    - No inputs or outputs.

---

### 3. Summary Table

| Node Name         | Node Type                             | Functional Role                           | Input Node(s)           | Output Node(s)      | Sticky Note                                                                                                                        |
|-------------------|-------------------------------------|-----------------------------------------|-------------------------|---------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Tally Form Response| Webhook                             | Receives Tally form submissions         | n/a                     | Set Email           |                                                                                                                                   |
| Set Email         | Set                                 | Sets recipient email address             | Tally Form Response     | Qualify Lead        | ## Set your email<br>Insert the email that will receive the LLM output                                                           |
| Qwen3-07-25       | Langchain LLM OpenRouter             | Provides AI model "qwen3-235b-a22b-07-25" | Qualify Lead (AI model) | Qualify Lead        |                                                                                                                                   |
| Gemini 2.5 pro    | Langchain LLM OpenRouter             | Provides AI model "google/gemini-2.5-pro" | Qualify Lead (AI model) | Qualify Lead        |                                                                                                                                   |
| Qualify Lead      | Chain LLM                           | Generates lead qualification report     | Set Email, Qwen3-07-25, Gemini 2.5 pro | Send a message     |                                                                                                                                   |
| Send a message    | Gmail                              | Sends email with AI-generated report    | Qualify Lead            | n/a                 |                                                                                                                                   |
| Sticky Note       | Sticky Note                        | Instruction: Set your email              | n/a                     | n/a                 | ## Set your email<br>Insert the email that will receive the LLM output                                                           |
| Sticky Note1      | Sticky Note                        | Setup requirements and instructions      | n/a                     | n/a                 | ## Requirements<br>- Create a [Tally](https://tally.cello.so/LEr7LHMwPcG) account<br>- Create a form<br>- Paste your n8n webhook URL into the Tally form's integrations tab.<br>- Edit your email in the Set Email node in n8n that will receive the LLM output<br>- Connect your desired ai model. We are using [OpenRouter](https://openrouter.ai) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Tally Form Response"**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Unique string (e.g., `bf493c41-2f48-4f67-8294-8ab73ddb84f4`)  
   - Purpose: Receive POST requests from Tally form integration.  
   - Ensure webhook URL is publicly accessible and configure in Tally form integrations.

2. **Create Set Node: "Set Email"**  
   - Type: Set  
   - Add String field: `toEmail`  
   - Value: Replace `"insert-your-email"` with the actual recipient email address to receive reports.  
   - Connect input from "Tally Form Response".

3. **Create Langchain Chain LLM Node: "Qualify Lead"**  
   - Type: Chain LLM (Langchain)  
   - Text parameter: Construct prompt text with dynamic expressions referencing Tally form fields, e.g.:  
     ```
     Company Name: {{ $('Tally Form Response').item.json.body.data.fields[0].value }}
     Full Name: {{ $('Tally Form Response').item.json.body.data.fields[1].value }}
     ... (and so on for all fields)
     ```
   - Include detailed instructions, security rules, and output format as per original prompt.  
   - Configure batching as appropriate (default empty).  
   - Connect input from "Set Email".  
   - Add two AI language model providers:
     - Node: "Qwen3-07-25"  
       - Type: Langchain LLM OpenRouter  
       - Model: `qwen/qwen3-235b-a22b-07-25`  
       - Credentials: OpenRouter API key  
     - Node: "Gemini 2.5 pro"  
       - Type: Langchain LLM OpenRouter  
       - Model: `google/gemini-2.5-pro`  
       - Credentials: OpenRouter API key  
   - Attach these model nodes as AI languageModel providers for "Qualify Lead".

4. **Create Gmail Node: "Send a message"**  
   - Type: Gmail  
   - Credentials: Configure Gmail OAuth2 credentials with send email permissions.  
   - Send To: Expression `={{ $('Set Email').item.json.toEmail }}`  
   - Subject: `"New form submission"`  
   - Message: Expression `={{ $json.text }}` (AI-generated report text)  
   - Email type: Plain text, no attribution.  
   - Connect input from "Qualify Lead".

5. **Add Sticky Notes for User Instructions**  
   - Create a Sticky Note near "Set Email" node:  
     Content:  
     ```
     ## Set your email
     Insert the email that will receive the LLM output
     ```  
   - Create another Sticky Note for overall requirements:  
     Content:  
     ```
     ## Requirements
     - Create a [Tally](https://tally.cello.so/LEr7LHMwPcG) account
     - Create a form
     - Paste your n8n webhook production url into the Tally form's integrations tab.
     - Edit your email in the Set Email node in n8n that will receive the LLM output
     - Connect your desired ai model. We are using [OpenRouter](https://openrouter.ai)
     ```

6. **Connect Nodes**  
   - "Tally Form Response" → "Set Email"  
   - "Set Email" → "Qualify Lead"  
   - "Qwen3-07-25" and "Gemini 2.5 pro" configured as language models linked to "Qualify Lead"  
   - "Qualify Lead" → "Send a message"

7. **Credential Setup**  
   - Obtain and configure OpenRouter API credentials for AI nodes.  
   - Obtain and configure Gmail OAuth2 credentials for sending emails.  
   - Ensure OAuth tokens are valid and have required scopes.

8. **Testing and Validation**  
   - Submit a test form via Tally to trigger the webhook.  
   - Verify AI-generated report is emailed to the configured address.  
   - Monitor logs for errors (e.g., AI model failures, email send failures).  
   - Adjust prompt or email address as necessary.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| The workflow requires a Tally account and form configured to send webhook POST requests to n8n.                                                                 | [Tally Forms](https://tally.cello.so/LEr7LHMwPcG)                                                              |
| The AI models used are accessed via OpenRouter, an open platform for AI model APIs.                                                                               | [OpenRouter](https://openrouter.ai)                                                                             |
| Gmail node requires OAuth2 credentials properly configured with send permissions.                                                                                 | n8n Gmail node documentation                                                                                     |
| Be cautious of prompt injection risks; the workflow enforces input field-only usage and professional output via prompt security rules.                           | Implemented in "Qualify Lead" node prompt                                                                        |
| The workflow example includes French labels from Tally form fields; ensure field indexing in prompt matches your specific form configuration.                    | Customize prompt expressions accordingly                                                                         |
| For large-scale deployment, monitor API usage quotas and email sending limits to avoid failures.                                                                 | OpenRouter and Gmail API rate limits                                                                              |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created using n8n, an integration and automation tool. This process strictly complies with current content policies and contains no illegal, offensive, or protected material. All handled data is legal and public.