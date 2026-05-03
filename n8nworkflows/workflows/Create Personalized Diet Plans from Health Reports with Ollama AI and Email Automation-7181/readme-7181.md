Create Personalized Diet Plans from Health Reports with Ollama AI and Email Automation

https://n8nworkflows.xyz/workflows/create-personalized-diet-plans-from-health-reports-with-ollama-ai-and-email-automation-7181


# Create Personalized Diet Plans from Health Reports with Ollama AI and Email Automation

### 1. Workflow Overview

This workflow automates the creation and delivery of personalized diet plans based on incoming health reports received via email. It targets nutritionists, healthcare providers, or wellness services aiming to offer custom diet advice derived from medical history or lab data. The workflow is structured into four main logical blocks:

- **1.1 Input Reception:** Receiving and extracting health report data from incoming emails via IMAP.
- **1.2 AI Processing:** Sending the extracted health data to an AI-powered nutrition model (Ollama) to generate a personalized diet plan.
- **1.3 Timing Control:** Introducing a controlled wait before sending the final diet plan, enabling asynchronous processing or user timing preferences.
- **1.4 Output Delivery:** Sending the finalized personalized diet plan to the user’s email via SMTP with an attached PDF.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block listens for new emails containing health or lab report data. It acts as the workflow’s trigger and extracts the raw data required for AI analysis.

- **Nodes Involved:**  
  - Receive Health Report Email

- **Node Details:**

  - **Receive Health Report Email**  
    - **Type:** Email Read (IMAP)  
    - **Role:** Triggers the workflow upon receiving a new email; reads email content and attachments.  
    - **Configuration:** Uses configured IMAP credentials to connect to the mailbox. No specific filters or folder restrictions mentioned, implying it reads all incoming emails.  
    - **Expressions/Variables:** The raw email content (including body and attachments) is passed forward as JSON.  
    - **Input/Output:** No input (trigger node). Outputs full email data to the next node.  
    - **Version:** n8n standard emailReadImap node version 2.  
    - **Potential Failures:**  
      - Authentication errors if IMAP credentials are invalid.  
      - Network or server timeouts.  
      - Parsing errors if email format is unexpected or corrupted.  
    - **Sub-Workflow:** None.

#### 2.2 AI Processing

- **Overview:**  
  This block processes the extracted health report data using an AI nutrition engine to generate a personalized diet plan tailored to the user’s medical history and dietary needs.

- **Nodes Involved:**  
  - Process Report with AI Nutrition Engine  
  - AI Nutrition Model

- **Node Details:**

  - **Process Report with AI Nutrition Engine**  
    - **Type:** LangChain LLM Chain  
    - **Role:** Sends a prompt containing health report data to an AI language model for detailed analysis and diet plan generation.  
    - **Configuration:**  
      - The prompt instructs the AI to analyze the health data and generate a detailed diet plan including meal suggestions, portion sizes, and nutritional goals.  
      - Uses an expression to inject the raw health report content from the email JSON (`{{json.content}}`).  
    - **Input/Output:** Receives email content from the IMAP node; sends prompt data to the AI Nutrition Model node.  
    - **Version:** 1.7 (LangChain node)  
    - **Potential Failures:**  
      - Expression errors if `json.content` is missing or malformed.  
      - AI service errors such as quota limits or API failures.  
    - **Sub-Workflow:** None.

  - **AI Nutrition Model**  
    - **Type:** Ollama AI Language Model  
    - **Role:** Executes the AI prompt and returns the personalized diet plan output.  
    - **Configuration:**  
      - Uses Ollama API credentials.  
      - No custom options specified, implying default model parameters.  
    - **Input/Output:** Receives prompt from the LangChain node; sends AI-generated text back to the LangChain node output.  
    - **Version:** 1  
    - **Potential Failures:**  
      - Authentication or API connection errors with Ollama.  
      - Timeout or response errors from the AI model.  
    - **Sub-Workflow:** None.

#### 2.3 Timing Control

- **Overview:**  
  Introduces a wait period before sending the final diet plan email, potentially to allow manual review or to comply with timing constraints.

- **Nodes Involved:**  
  - Wait Before Sending Plan

- **Node Details:**

  - **Wait Before Sending Plan**  
    - **Type:** Wait node  
    - **Role:** Pauses the workflow’s execution after AI processing before proceeding to email sending.  
    - **Configuration:** No specific wait duration configured (defaults or webhook-controlled wait). The presence of a webhook ID suggests it may pause until externally triggered.  
    - **Input/Output:** Receives AI-processed data; outputs data for the final email send node.  
    - **Version:** 1.1  
    - **Potential Failures:**  
      - Workflow timeout if the wait is indefinite.  
      - Webhook triggering errors if external trigger is not received.  
    - **Sub-Workflow:** None.

#### 2.4 Output Delivery

- **Overview:**  
  Sends the personalized diet plan as a PDF attachment via email to the user’s specified email address.

- **Nodes Involved:**  
  - Send Personalized Diet Plan

- **Node Details:**

  - **Send Personalized Diet Plan**  
    - **Type:** Email Send (SMTP)  
    - **Role:** Delivers the personalized diet plan to the user’s email inbox.  
    - **Configuration:**  
      - Email subject: "Your Personalized Diet Plan"  
      - Email body: A friendly message referencing the attached diet plan PDF.  
      - Recipient email dynamically set from webhook JSON userEmail (`{{$node['Webhook'].json['userEmail']}}`).  
      - Sender email is statically set as `your-email@domain.com`.  
      - Attachment specified dynamically from a previous node’s file path (`{{$node['Generate PDF'].json['filePath']}}`), although note the workflow JSON does not show a "Generate PDF" node, implying a missing or external step.  
      - Uses configured SMTP credentials.  
    - **Input/Output:** Receives data from the wait node; sends email externally; no output.  
    - **Version:** 1  
    - **Potential Failures:**  
      - SMTP authentication failures.  
      - Missing or invalid email addresses.  
      - Missing attachment file if the PDF was not generated properly.  
    - **Sub-Workflow:** None.

#### 2.5 Additional Notes

- **Sticky Note**  
  - Summarizes the workflow’s key node roles and logical flow for user clarity.  
  - Covers the entire process from email trigger to sending the diet plan email.  
  - Provides a high-level description of each step, useful for onboarding or handover.

---

### 3. Summary Table

| Node Name                     | Node Type                   | Functional Role                          | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                             |
|-------------------------------|-----------------------------|----------------------------------------|--------------------------------|---------------------------------|-------------------------------------------------------------------------|
| Receive Health Report Email    | Email Read (IMAP)            | Trigger workflow on new health report email | None                           | Process Report with AI Nutrition Engine | Describes overall email-triggered workflow steps and logic             |
| Process Report with AI Nutrition Engine | LangChain LLM Chain          | Creates AI prompt with health data for diet plan generation | Receive Health Report Email     | Wait Before Sending Plan          | Describes overall email-triggered workflow steps and logic             |
| AI Nutrition Model             | Ollama AI Language Model     | Executes AI prompt to generate diet plan text | Process Report with AI Nutrition Engine | Process Report with AI Nutrition Engine (ai_languageModel) | Describes overall email-triggered workflow steps and logic             |
| Wait Before Sending Plan       | Wait                        | Pauses workflow before sending diet plan email | Process Report with AI Nutrition Engine | Send Personalized Diet Plan       | Describes overall email-triggered workflow steps and logic             |
| Send Personalized Diet Plan    | Email Send (SMTP)           | Sends finalized diet plan PDF to user email | Wait Before Sending Plan         | None                            | Sends the PDF diet plan to the user via email                          |
| Sticky Note                   | Sticky Note                 | Provides workflow overview and node descriptions | None                           | None                            | Describes overall email-triggered workflow steps and logic             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the IMAP Email Trigger Node:**  
   - Add an **Email Read (IMAP)** node named "Receive Health Report Email".  
   - Configure it with valid IMAP credentials to access the mailbox receiving health reports.  
   - Leave options default to read all incoming emails.  
   - This node triggers the workflow on new email received.

2. **Add LangChain LLM Chain Node:**  
   - Add a **LangChain Chain LLM** node named "Process Report with AI Nutrition Engine".  
   - Connect it to the "Receive Health Report Email" node.  
   - In the "Messages" parameter, create a message with the following text (use expression to insert email content):  
     ```
     Analyze the following health details or lab report data and generate a personalized diet plan tailored to the user's medical history, dietary restrictions, and nutritional needs:

     {{json.content}}

     Provide a detailed diet plan including daily meal suggestions, portion sizes, and nutritional goals.
     ```  
   - No batching or advanced options need to be changed.

3. **Add Ollama AI Language Model Node:**  
   - Add an **Ollama** node named "AI Nutrition Model".  
   - Configure it with valid Ollama API credentials.  
   - Connect the LangChain node’s AI language model output to this node’s input.  
   - Use default model options.

4. **Connect the AI Nutrition Model back to LangChain node:**  
   - Ensure the AI output is routed back correctly as LangChain expects (this is usually handled automatically by the LangChain node configuration).

5. **Add a Wait Node:**  
   - Add a **Wait** node named "Wait Before Sending Plan".  
   - Connect the output of the LangChain node ("Process Report with AI Nutrition Engine") to this node.  
   - Configure the wait duration as needed (if indefinite wait is needed, configure webhook ID). Otherwise, set a fixed wait time.

6. **Add Email Send Node:**  
   - Add an **Email Send (SMTP)** node named "Send Personalized Diet Plan".  
   - Connect the wait node output to this node.  
   - Configure SMTP credentials with a valid sending email account.  
   - Set the recipient email dynamically using an incoming trigger variable, e.g., `{{$node['Webhook'].json['userEmail']}}` (note: ensure userEmail is captured somewhere in the workflow or replace with actual email).  
   - Set sender email as your email address.  
   - Compose email subject as "Your Personalized Diet Plan".  
   - Compose email body with a friendly message referencing the attached diet plan.  
   - Attach the diet plan PDF file path (requires a "Generate PDF" node or external process to create the PDF file; this step is not present in the JSON and must be added).  
   - If no PDF generation is present, either add a PDF generation node or change attachment handling accordingly.

7. **(Optional) Add PDF Generation Node:**  
   - Since the workflow references `Generate PDF` node output for the attachment, add a PDF generation node (e.g., "HTML to PDF") before the wait node or before sending email.  
   - Configure it to convert the AI diet plan text into a PDF file.  
   - Pass the generated file path to the email node attachment field.

8. **Workflow Activation and Testing:**  
   - Activate the workflow.  
   - Test by sending a health report email to the IMAP inbox.  
   - Monitor logs for AI processing and email sending success.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                     |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------|
| Workflow triggers on receipt of emails containing health or lab reports to automate diet creation. | Workflow purpose summary                            |
| Uses Ollama AI to generate personalized diet plans based on detailed health data analysis.          | AI integration details                              |
| Email sending includes PDF attachment assumed to be generated within or outside this workflow.      | Requires PDF generation step (missing in JSON)    |
| For IMAP and SMTP credentials, ensure proper OAuth2 or app-specific passwords per provider rules.  | Credential setup best practices                     |
| Sticky note in workflow summarizes node roles for quick reference.                                  | Included in workflow JSON as a sticky note node    |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. All data processed are legal and publicly available, and the workflow complies with content policies without generating illegal or offensive content.