Automated Error Notifications with Optional GPT-4o Diagnostics via Email

https://n8nworkflows.xyz/workflows/automated-error-notifications-with-optional-gpt-4o-diagnostics-via-email-11507


# Automated Error Notifications with Optional GPT-4o Diagnostics via Email

### 1. Workflow Overview

This workflow automates error notification for any failed n8n workflow execution within an instance by sending a formatted email alert. It targets developers, operators, or teams needing immediate visibility on workflow failures with an optional AI-assisted diagnostic feature. The workflow logically organizes into these blocks:

- **1.1 Error Capture and Configuration Setup:** Captures error details when any workflow fails and sets configurable email parameters including sender, recipient, and whether to invoke AI analysis.

- **1.2 Conditional AI Analysis:** Based on user configuration, optionally sends the error details to an OpenAI GPT-4o-based node to receive a severity level and quick resolution suggestion.

- **1.3 Email Formatting:** Prepares a rich HTML email body that includes error details and, if available, AI diagnostic results.

- **1.4 Email Dispatch:** Sends the composed email via SMTP credentials.

- **1.5 Informational Sticky Notes:** Provide usage instructions, overview, and setup guidance for users.

---

### 2. Block-by-Block Analysis

#### 2.1 Error Capture and Configuration Setup

- **Overview:**  
This block triggers when any workflow error occurs in the n8n instance, capturing essential context and setting email parameters and AI usage preferences.

- **Nodes Involved:**  
  - Error Trigger  
  - Config - Set Fields

- **Node Details:**

  - **Error Trigger**  
    - Type: `Error Trigger` (system node)  
    - Role: Listens globally for any workflow execution error in the n8n instance.  
    - Configuration: Default, no custom parameters.  
    - Key Data: Provides error context such as workflow name, workflow ID, last node executed, run ID, error message, and execution URL.  
    - Connections: Outputs error JSON to "Config - Set Fields".  
    - Edge Cases: No input; failure possible if n8n instance is offline or error event not emitted.  
    - Version: v1

  - **Config - Set Fields**  
    - Type: `Set`  
    - Role: Defines static and dynamic email parameters and a toggle for AI analysis.  
    - Configuration:  
      - `fromEmail`: "automate@xyz.com" (sender email)  
      - `toEmail`: "xyz@gmail.com" (recipient email)  
      - `emailSubject`: Dynamic subject including workflow name (e.g., "🤖 N8N Workflow Error Alert: [Workflow Name]")  
      - `AnalyzeErrorWithAI`: Boolean flag (default false) to control AI analysis usage  
    - Expressions: Uses `{{$node['Error Trigger'].json.workflow.name}}` inside the subject line.  
    - Input: From "Error Trigger"  
    - Output: To "Use AI Analysis?"  
    - Edge Cases: Incorrect email formats or missing values may cause SMTP send failures. Misconfigured toggle disables AI path.  
    - Version: v3.4

---

#### 2.2 Conditional AI Analysis

- **Overview:**  
Determines if the workflow error should be analyzed by the AI node based on the toggle; routes the flow accordingly.

- **Nodes Involved:**  
  - Use AI Analysis?  
  - Analyze Error with AI (conditional)

- **Node Details:**

  - **Use AI Analysis?**  
    - Type: `If`  
    - Role: Branches workflow based on the boolean `AnalyzeErrorWithAI` value from Config node.  
    - Configuration: Checks if `AnalyzeErrorWithAI` is `true`.  
    - Input: From "Config - Set Fields"  
    - Output:  
      - True branch: To "Analyze Error with AI"  
      - False branch: To "Format Email Body" directly (skips AI)  
    - Edge Cases: Expression failures if field missing or malformed boolean.  
    - Version: v2.2

  - **Analyze Error with AI**  
    - Type: `OpenAI (LangChain)` node  
    - Role: Sends error details to GPT-4o-mini model for diagnostic JSON response including severity and quick resolution.  
    - Configuration:  
      - Model: "gpt-4o-mini"  
      - Prompt includes: workflow name, workflow ID, node name, run ID, error message.  
      - System prompt instructs AI to respond with strict JSON object containing "severity_level" and "quick_resolution".  
    - Expressions: Dynamic prompt content using error data from "Error Trigger".  
    - Credentials: Requires configured OpenAI API credentials.  
    - Input: From "Use AI Analysis?" true branch  
    - Output: To "Format Email Body"  
    - Edge Cases:  
      - API authentication errors  
      - Timeout or rate limits from OpenAI  
      - Malformed or missing error data causing AI prompt issues  
      - Incorrect or unexpected AI response breaking JSON parsing  
    - Version: v2.1

---

#### 2.3 Email Formatting

- **Overview:**  
Prepares the HTML email content incorporating error details and optional AI analysis results, ensuring a clean, styled alert message.

- **Nodes Involved:**  
  - Format Email Body

- **Node Details:**

  - **Format Email Body**  
    - Type: `Set`  
    - Role: Generates a complete HTML email body using a template with embedded dynamic data references.  
    - Configuration:  
      - Sets `emailBody` field with an HTML string that includes:  
        - Workflow name, ID, last node executed, run ID  
        - Severity level and quick resolution if AI analysis was executed  
        - Error message  
        - A styled call-to-action button linking to the failed workflow run URL  
      - Uses expressions referencing both "Error Trigger" and (optionally) "Analyze Error with AI" nodes.  
    - Input: From either "Analyze Error with AI" or "Use AI Analysis?" false branch  
    - Output: To "Send email"  
    - Edge Cases:  
      - If AI analysis node is skipped, severity and quick resolution fields are empty gracefully.  
      - Expression failures if referenced nodes data missing or malformed.  
    - Version: v3.4

---

#### 2.4 Email Dispatch

- **Overview:**  
Sends the formatted email alert to the configured recipient(s) using SMTP credentials.

- **Nodes Involved:**  
  - Send email

- **Node Details:**

  - **Send email**  
    - Type: `EmailSend`  
    - Role: Delivers the alert email.  
    - Configuration:  
      - `subject`: Dynamic, set from "Config - Set Fields" emailSubject  
      - `toEmail`: From Config node  
      - `fromEmail`: From Config node  
      - `html`: The email body from "Format Email Body"  
      - SMTP credentials: Preconfigured with SMTP account (e.g., company mail server)  
    - Input: From "Format Email Body"  
    - Output: None (terminal node)  
    - Edge Cases:  
      - SMTP authentication or connectivity errors  
      - Invalid email addresses causing delivery failure  
      - Large email body size exceeding limits  
    - Version: v2.1

---

#### 2.5 Informational Sticky Notes

- **Overview:**  
Provide detailed instructions and overview for users on the workflow’s purpose, usage, and setup.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1

- **Node Details:**

  - **Sticky Note**  
    - Type: `Sticky Note`  
    - Role: Describes workflow overview, logic, usage, and customization options.  
    - Content Highlights:  
      - Workflow purpose: Automated error notifications with optional AI diagnostics  
      - Stepwise logic explanation  
      - Setup instructions including email and OpenAI credentials  
      - Customization suggestions for alert channels  
    - Position: Top-left for prominent visibility

  - **Sticky Note1**  
    - Type: `Sticky Note`  
    - Role: Provides a link to a YouTube video explaining how to use the workflow.  
    - Content: "## How to Use This Workflow\n@[youtube](nzEApie96xk)"  
    - Position: Adjacent to Error Trigger node  

---

### 3. Summary Table

| Node Name           | Node Type            | Functional Role                      | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                                           |
|---------------------|----------------------|------------------------------------|-----------------------|------------------------|-----------------------------------------------------------------------------------------------------------------------|
| Error Trigger       | Error Trigger        | Captures any workflow error event  | None                  | Config - Set Fields     | ## How to Use This Workflow @[youtube](nzEApie96xk)                                                                   |
| Config - Set Fields | Set                  | Sets email parameters and AI toggle| Error Trigger         | Use AI Analysis?        | ## Overview — Error Alerts with Optional AI Insights... (full detailed note on error alert workflow logic and setup)  |
| Use AI Analysis?    | If                   | Branches flow based on AI toggle   | Config - Set Fields    | Analyze Error with AI, Format Email Body |                                                                                                                       |
| Analyze Error with AI| OpenAI (LangChain)   | Sends error info to GPT-4o for analysis | Use AI Analysis? (true) | Format Email Body       |                                                                                                                       |
| Format Email Body   | Set                  | Formats HTML email content          | Analyze Error with AI or Use AI Analysis? (false) | Send email              |                                                                                                                       |
| Send email          | EmailSend            | Sends formatted email alert         | Format Email Body      | None                   |                                                                                                                       |
| Sticky Note         | Sticky Note          | Workflow overview and setup guide   | None                  | None                   | ## Overview — Error Alerts with Optional AI Insights...                                                              |
| Sticky Note1        | Sticky Note          | Link to usage video                 | None                  | None                   | ## How to Use This Workflow @[youtube](nzEApie96xk)                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Error Trigger" node**  
   - Type: `Error Trigger`  
   - Purpose: Captures any workflow error globally.  
   - No parameters needed.  

2. **Create "Config - Set Fields" node**  
   - Type: `Set`  
   - Parameters to set:  
     - `fromEmail`: "automate@xyz.com"  
     - `toEmail`: "xyz@gmail.com"  
     - `emailSubject`: Set as expression: `🤖 N8N Workflow Error Alert: {{$node["Error Trigger"].json.workflow.name}}`  
     - `AnalyzeErrorWithAI`: Boolean, default `false`  
   - Connect input from "Error Trigger".  

3. **Create "Use AI Analysis?" node**  
   - Type: `If` node  
   - Condition: Check if `{{$node["Config - Set Fields"].json.AnalyzeErrorWithAI}}` is true (boolean check).  
   - Connect input from "Config - Set Fields".  

4. **Create "Analyze Error with AI" node**  
   - Type: `OpenAI (LangChain)`  
   - Credentials: Configure and select your OpenAI API credentials.  
   - Model: Select "gpt-4o-mini".  
   - System prompt:  
     ```
     You are an expert automation engineer. 
     Always respond with a RAW JSON object without wrapping it in quotes. 
     Never escape characters, never add \n, never format JSON as a string.
     Never include explanations, markdown, commentary, or prose.
     return ONLY valid JSON that includes:
     - severity level
     - a detailed (not more than 100 words) suggested configuration/change
     Do NOT include explanations, prose, markdown, commentary, or any text before or after the JSON.
     ```
   - User prompt:  
     ```
     Analyze the following workflow error and return ONLY a valid JSON object (do not include \n etc...) with this exact structure:

     {
       "severity_level": "",
       "quick_resolution": ""
     }

     ### Context:
     Workflow Name: {{ $node['Error Trigger'].json.workflow.name || "Unknown" }}
     Workflow ID: {{ $node['Error Trigger'].json.workflow.id || "N/A" }}
     Node Name: {{ $node['Error Trigger'].json.execution.lastNodeExecuted || "Unknown" }}
     Run ID: {{ $node['Error Trigger'].json.execution.id || "N/A" }}
     Error Message: {{ $node['Error Trigger'].json.execution.error.message }}

     ### Requirements:
     1. "severity_level" must be one of: Low, Medium, High, Critical.
     2. "quick_resolution" must be a detailed (not more than 100 words), practical fix the user can apply.
     3. The response MUST be valid JSON and nothing else.

     Return the JSON object directly, not as a string. Do not escape characters.
     ```
   - Connect input from "Use AI Analysis?" true branch.  

5. **Create "Format Email Body" node**  
   - Type: `Set`  
   - Set a single field `emailBody` with the entire HTML template as a string. Within the HTML, embed expressions:  
     - Workflow details from "Error Trigger" node (e.g., workflow name, ID, last node executed, run ID, error message).  
     - Conditionally include severity level and quick resolution from "Analyze Error with AI" node only if executed.  
     - Include a call-to-action button linked to the workflow execution URL.  
   - Connect input from:  
     - "Analyze Error with AI" node (if AI used)  
     - "Use AI Analysis?" false branch (if AI skipped)  

6. **Create "Send email" node**  
   - Type: `EmailSend`  
   - Credentials: Configure SMTP credentials (e.g., company SMTP account).  
   - Parameters:  
     - `fromEmail`: Expression from "Config - Set Fields" (`{{$node["Config - Set Fields"].json.fromEmail}}`)  
     - `toEmail`: Expression from "Config - Set Fields" (`{{$node["Config - Set Fields"].json.toEmail}}`)  
     - `subject`: Expression from "Config - Set Fields" (`{{$node["Config - Set Fields"].json.emailSubject}}`)  
     - `html`: Expression from "Format Email Body" (`{{$node["Format Email Body"].json.emailBody}}`)  
   - Connect input from "Format Email Body".  

7. **Create Sticky Notes** (optional but recommended)  
   - Add a sticky note near top-left describing workflow overview, usage, and setup instructions.  
   - Add a sticky note near the "Error Trigger" with usage video link: `@[youtube](nzEApie96xk)`.  

8. **Connect all nodes as per the described flow:**  
   - Error Trigger → Config - Set Fields → Use AI Analysis?  
   - Use AI Analysis? → True → Analyze Error with AI → Format Email Body → Send email  
   - Use AI Analysis? → False → Format Email Body → Send email  

9. **Activate the workflow** to start monitoring all workflows for errors and sending alerts accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                               | Context or Link                         |
|----------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| Workflow sends email alerts on any workflow failure with optional GPT-4o AI diagnostics for severity and quick fix advice. | Workflow overview and purpose          |
| Setup requires SMTP credentials for email sending and optionally OpenAI API credentials for AI analysis.                    | Credential setup instructions          |
| Customizable email template uses modern HTML with embedded variables for dynamic content and styling.                        | Email formatting node content          |
| YouTube usage video link: [https://youtu.be/nzEApie96xk](https://youtu.be/nzEApie96xk)                                       | Sticky Note1 content                   |
| AI node strictly expects JSON-only response, no markdown or extraneous text; failure to comply can break parsing.            | Analyze Error with AI node instructions|
| Can be extended to send alerts to other channels (Slack, Discord, Telegram) by adding parallel notification nodes.           | Customization suggestion               |

---

**Disclaimer:**  
The provided text is extracted solely from an automated n8n workflow. It strictly complies with content policies, contains no illegal or offensive elements, and handles only legal and public data.