Simplify Medical Lab Reports with Ollama AI and Email Delivery System

https://n8nworkflows.xyz/workflows/simplify-medical-lab-reports-with-ollama-ai-and-email-delivery-system-7274


# Simplify Medical Lab Reports with Ollama AI and Email Delivery System

### 1. Workflow Overview

This workflow automates the process of simplifying medical lab reports received via email and delivering patient-friendly versions back to the patients. It targets healthcare providers or medical labs aiming to enhance patient communication by converting complex medical terminology into easy-to-understand language with actionable advice.

The workflow is logically grouped into the following blocks:

**1.1 Input Reception**  
- Monitors an email inbox for incoming lab reports, specifically looking for emails with PDF attachments.

**1.2 Data Extraction and Preparation**  
- Extracts the content of the lab report from email attachments or fallback content within the email and extracts the patient’s email address.

**1.3 AI Processing**  
- Uses an Ollama AI medical language model to analyze and simplify the lab report content into a plain-language summary including explanations, key findings, and precautions.

**1.4 Formatting and Output Generation**  
- Formats the AI-generated simplified report into a styled HTML email and a WhatsApp-friendly plain text message.

**1.5 Email Delivery**  
- Sends the formatted simplified lab report to the patient’s email address.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block continuously monitors a configured email inbox using IMAP for new lab report emails, including those with PDF attachments.

**Nodes Involved:**  
- Lab Report Email Trigger

**Node Details:**

- **Lab Report Email Trigger**  
  - Type: Email Read (IMAP)  
  - Role: Watches the email inbox for incoming messages potentially containing lab reports.  
  - Configuration: Uses IMAP credentials to connect; default options without filters, implying it reads all incoming emails.  
  - Inputs: None (trigger node)  
  - Outputs: Email data including attachments, sender, subject, and text content.  
  - Potential Failures: Connection/authentication errors with IMAP server, no new emails triggering, attachment format inconsistencies.  
  - Notes: Email filter configuration recommended externally to limit to lab reports only.

---

#### 2.2 Data Extraction and Preparation

**Overview:**  
Extracts the relevant lab report content from email attachments (PDF preferred) or email body fallback and retrieves the patient’s email address for later use.

**Nodes Involved:**  
- Extract PDF Content

**Node Details:**

- **Extract PDF Content**  
  - Type: Code (JavaScript)  
  - Role: Parses incoming email to find PDF attachments. If found, attempts to extract content (currently a placeholder using email text as fallback). Extracts sender’s email as patient email.  
  - Configuration: Custom JavaScript logic checks for PDF attachments by content type or filename; uses email text/plain or html if PDF extraction not implemented.  
  - Inputs: Email data from Lab Report Email Trigger  
  - Outputs: JSON object with `patient_email`, `report_content`, `subject`, and `received_date`.  
  - Edge Cases: Missing attachments, non-PDF attachments, malformed email sender address, empty email body. PDF parsing is a stub — real parsing library integration recommended for production.  
  - Version Dependencies: None specific, but assumes n8n v2+ for code node features.

---

#### 2.3 AI Processing

**Overview:**  
Uses Ollama AI’s llama3.2-16000 model via Langchain integration to simplify the medical lab report text into a patient-friendly summary, explanations, key findings, and advice.

**Nodes Involved:**  
- Medical AI Model (Ollama)  
- AI Report Simplifier

**Node Details:**

- **Medical AI Model**  
  - Type: Langchain LM Chat Ollama  
  - Role: Provides access to the Ollama llama3.2-16000 language model for conversational AI processing.  
  - Configuration: Uses Ollama API credentials, no additional options specified.  
  - Inputs: None directly; linked to AI Report Simplifier as language model resource.  
  - Outputs: Language model capability for downstream nodes.  
  - Failures: API authentication issues, model unavailability, timeout on large inputs.

- **AI Report Simplifier**  
  - Type: Langchain Agent  
  - Role: Sends the extracted lab report content to the AI model with a detailed system prompt to simplify and explain the report in easy language, providing summaries, normal/abnormal result explanations, key findings, and precautionary advice.  
  - Configuration:  
    - Input text: report content from Extract PDF Content node  
    - Prompt: Extensive system message instructing the AI to produce structured, jargon-free, reassuring explanations ending with a disclaimer to consult a healthcare provider.  
  - Inputs: Report content JSON from Extract PDF Content node  
  - Outputs: Simplified AI-generated report text under `output` attribute  
  - Failures: Expression evaluation errors, empty or malformed input text, AI model errors or rate limits, incomplete response formatting.

---

#### 2.4 Formatting and Output Generation

**Overview:**  
Transforms the AI-generated simplified text into a visually styled HTML email and a plain text WhatsApp-friendly message, including metadata like report date and original subject.

**Nodes Involved:**  
- Format Report Response

**Node Details:**

- **Format Report Response**  
  - Type: Code (JavaScript)  
  - Role: Formats the simplified report with HTML styling for email and creates a plain text message suitable for WhatsApp. Includes report date, original email subject, and next steps advice in the layout.  
  - Configuration:  
    - Uses original extracted data and AI output to build content  
    - HTML uses inline CSS with color coding and boxes for clarity and branding  
    - Plain text converts markdown-style emphasis to asterisks for WhatsApp compatibility  
  - Inputs: AI Report Simplifier output and Extract PDF Content data  
  - Outputs: JSON with `patient_email`, formatted `html_content`, `plain_content`, `whatsapp_message`, and email `subject`.  
  - Edge Cases: Missing AI output, formatting errors in text replacements, date localization issues.

---

#### 2.5 Email Delivery

**Overview:**  
Sends the final formatted simplified lab report to the patient’s email address using SMTP.

**Nodes Involved:**  
- Send Simplified Report

**Node Details:**

- **Send Simplified Report**  
  - Type: Email Send (SMTP)  
  - Role: Sends an email with the formatted HTML and plain text content to the patient’s email extracted earlier.  
  - Configuration:  
    - `To`: Patient’s email from JSON  
    - `From`: Fixed sender address lab-reports@yourmedical.com  
    - Subject: Customized with original report subject  
    - Credentials: SMTP credentials configured for sending email  
  - Inputs: Formatted content from Format Report Response  
  - Outputs: Status of email sending  
  - Failures: SMTP authentication errors, invalid recipient email, email sending timeouts, spam filtering issues.

---

### 3. Summary Table

| Node Name              | Node Type                    | Functional Role                     | Input Node(s)          | Output Node(s)          | Sticky Note                                                                                          |
|------------------------|------------------------------|-----------------------------------|-----------------------|-------------------------|----------------------------------------------------------------------------------------------------|
| Lab Report Email Trigger| Email Read (IMAP)             | Monitor email inbox for lab reports| None                  | Extract PDF Content      |                                                                                                    |
| Extract PDF Content     | Code                         | Extract and prepare lab report content and patient email | Lab Report Email Trigger | AI Report Simplifier    |                                                                                                    |
| Medical AI Model        | Langchain LM Chat Ollama     | Provide Ollama AI medical language model | None                  | AI Report Simplifier     |                                                                                                    |
| AI Report Simplifier    | Langchain Agent              | Simplify lab report text using AI | Medical AI Model, Extract PDF Content | Format Report Response |                                                                                                    |
| Format Report Response  | Code                         | Format simplified report for email and WhatsApp | AI Report Simplifier   | Send Simplified Report   |                                                                                                    |
| Send Simplified Report  | Email Send (SMTP)            | Send simplified report email to patient | Format Report Response | None                    |                                                                                                    |
| Workflow Overview       | Sticky Note                  | Workflow summary and purpose      | None                  | None                    | ## 🩺 AI Lab Report Simplifier<br>Streamlined 6-Node Workflow:<br>Automatic PDF processing, medical term simplification, precautionary advice, professional formatting, HIPAA-compliant handling |
| Setup Instructions      | Sticky Note                  | Setup requirements and instructions| None                  | None                    | ## ⚙️ Setup Guide:<br>Required: Email IMAP, Ollama AI, SMTP<br>Configure email filters, AI medical model, PDF parsing library, and test with samples |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Node: "Lab Report Email Trigger"**  
   - Type: Email Read (IMAP)  
   - Configure IMAP credentials with access to the lab report inbox.  
   - Set options as needed (default reads all emails).  
   - Position on canvas: Leftmost (e.g., -600, 360).  
   - No inputs (trigger node).

2. **Create Node: "Extract PDF Content"**  
   - Type: Code (JavaScript)  
   - Connect input from "Lab Report Email Trigger".  
   - Paste the provided JavaScript code that:  
     - Checks attachments for PDFs by content type or filename.  
     - Falls back to email text/plain or html if no PDF content extraction is done.  
     - Extracts sender’s email as patient_email.  
     - Outputs JSON with `patient_email`, `report_content`, `subject`, and `received_date`.  
   - Position near "Lab Report Email Trigger" (e.g., -380, 360).

3. **Create Node: "Medical AI Model"**  
   - Type: Langchain LM Chat Ollama  
   - Configure using Ollama API credentials.  
   - Select model: `llama3.2-16000:latest`.  
   - No additional options needed.  
   - Position: Left-mid section (e.g., -72, 580).  
   - No input connections; used as a resource node.

4. **Create Node: "AI Report Simplifier"**  
   - Type: Langchain Agent  
   - Connect input from "Extract PDF Content".  
   - Configure:  
     - Text input: Expression referencing `{{$json.report_content}}`.  
     - System message prompt: Use the detailed prompt instructing the AI to simplify medical lab reports into easy language with summaries, explanations, key findings, and precautions, ending with a disclaimer.  
   - Link the "Medical AI Model" node as the language model resource under `ai_languageModel` connection.  
   - Position: Center-left (e.g., -160, 360).

5. **Create Node: "Format Report Response"**  
   - Type: Code (JavaScript)  
   - Connect input from "AI Report Simplifier".  
   - Paste the JavaScript code that:  
     - Reads the original extracted data and AI output.  
     - Creates styled HTML content with inline CSS and a WhatsApp-friendly plain text message.  
     - Outputs JSON with `patient_email`, `html_content`, `plain_content`, `whatsapp_message`, and `subject`.  
   - Position: Center-right (e.g., 216, 360).

6. **Create Node: "Send Simplified Report"**  
   - Type: Email Send (SMTP)  
   - Connect input from "Format Report Response".  
   - Configure SMTP credentials for sending emails.  
   - Set "To Email" field to `{{$json.patient_email}}`.  
   - Set "From Email" field to a fixed sender address, e.g., `lab-reports@yourmedical.com`.  
   - Set "Subject" field to a dynamic value, e.g., `Your Simplified Lab Report - {{$json.original_subject}}`.  
   - Use `html_content` for HTML body and `plain_content` for plain text.  
   - Position: Rightmost (e.g., 436, 360).

7. **Add Sticky Notes for Documentation (Optional but recommended):**  
   - "Workflow Overview" sticky note describing the entire workflow purpose and node roles.  
   - "Setup Instructions" sticky note listing required credentials, configuration tips, and input/output expectations.

8. **Connection Summary:**  
   - Lab Report Email Trigger → Extract PDF Content  
   - Extract PDF Content → AI Report Simplifier  
   - Medical AI Model → AI Report Simplifier (AI model resource)  
   - AI Report Simplifier → Format Report Response  
   - Format Report Response → Send Simplified Report

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                      |
|-----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Automatic PDF processing is stubbed; integrate a PDF parsing library (e.g., pdf-lib or pdf.js) for real extraction.   | Setup Instructions sticky note                       |
| Ollama AI model `llama3.2-16000:latest` is used; ensure API quota and availability.                                   | Medical AI Model node configuration                  |
| HIPAA-compliance implied for secure handling of patient data; configure credentials and environment securely.         | Workflow Overview sticky note                         |
| Always include disclaimers advising patients to consult healthcare providers despite AI simplification.              | AI Report Simplifier prompt and Format Report Response node |
| Test workflow with sample lab report emails containing PDF attachments for reliability before production deployment. | Setup Instructions sticky note                       |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.