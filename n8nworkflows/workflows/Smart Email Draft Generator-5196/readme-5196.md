Smart Email Draft Generator

https://n8nworkflows.xyz/workflows/smart-email-draft-generator-5196


# Smart Email Draft Generator

### 1. Workflow Overview

The **Smart Email Draft Generator** workflow automates the process of creating email draft replies by reading incoming emails via IMAP, generating AI-based reply content, and saving the drafts in a Gmail account. It targets users who want to streamline email response drafting using AI language models.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Reading new emails from an IMAP account.
- **1.2 AI Processing:** Generating a reply draft from the incoming email content using AI models.
- **1.3 Email Preparation:** Formatting the AI-generated text and setting email fields (from, subject, body).
- **1.4 Draft Saving:** Saving the prepared email as a draft in Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block monitors a designated IMAP email inbox and triggers the workflow whenever a new email is detected.

- **Nodes Involved:**  
  - Check New Email (IMAP)

- **Node Details:**  

  - **Check New Email (IMAP):**  
    - Type: Email Read (IMAP) node  
    - Role: Polls new emails from the configured IMAP inbox.  
    - Configuration: Uses stored IMAP credentials; default options with no custom filters.  
    - Expressions: None.  
    - Input: Workflow trigger (implicit).  
    - Output: Emits the full email JSON, including `from`, `subject`, and plaintext content (`textPlain`).  
    - Version: 2  
    - Edge Cases:  
      - IMAP authentication failures.  
      - No new emails (workflow does not proceed).  
      - Large emails or attachment handling is not configured (may affect performance).  
    - Sub-workflow: None.

#### 1.2 AI Processing

- **Overview:**  
  Generates a reply draft by passing the incoming email's plaintext content to AI language models. This uses two AI nodes in sequence, with the second node (`Custom AI Model`) connected as an alternative language model.

- **Nodes Involved:**  
  - Process Email with AI  
  - Custom AI Model

- **Node Details:**  

  - **Process Email with AI:**  
    - Type: LangChain LLM Chain node  
    - Role: Primary AI processing to generate reply text from the incoming email's plaintext.  
    - Configuration:  
      - Input text set via expression `={{ $json.textPlain }}` (extracts plaintext email body).  
      - Prompt template prepends:  
        ```
        this is email

        {{ $json.textPlain }}

        i need reply of this 
        ```  
      - PromptType: Defined prompt.  
    - Input: Email text from "Check New Email (IMAP)".  
    - Output: Generated reply text under `text` property.  
    - Version: 1.6  
    - Edge Cases:  
      - AI service errors (timeout, rate limits).  
      - Empty or malformed email text.  
    - Sub-workflow: None.

  - **Custom AI Model:**  
    - Type: LangChain Ollama LM node  
    - Role: Alternative/custom AI model (llama3.2-16000) to process the email text.  
    - Configuration:  
      - Model: `llama3.2-16000:latest`  
      - No extra options configured.  
      - Credentials: Ollama API key set.  
    - Input: Linked as AI languageModel output to "Process Email with AI" node.  
    - Output: AI-generated text (not further connected in this workflow; likely a fallback or for testing).  
    - Version: 1  
    - Edge Cases:  
      - Ollama API authentication errors.  
      - Model response latency or failures.  
    - Sub-workflow: None.

#### 1.3 Email Preparation

- **Overview:**  
  Formats the output of the AI-generated reply into an email draft structure, setting standard email fields for "from", "subject", and "text" body.

- **Nodes Involved:**  
  - Prepare Email Content

- **Node Details:**  

  - **Prepare Email Content:**  
    - Type: Set node  
    - Role: Assigns and formats email fields using expressions.  
    - Configuration:  
      - `from`: Extracted from the original email's `from` field (`={{ $('Check New Email (IMAP)').first().json.from }}`)  
      - `subject`: Prepends "Re: " to the original email subject (`=Re: {{ $('Check New Email (IMAP)').first().json.subject }}`)  
      - `text`: Takes the AI-generated reply text (`={{ $json.text }}`)  
    - Input: Output from "Process Email with AI" node.  
    - Output: Structured email draft JSON with `from`, `subject`, and `text`.  
    - Version: 3.4  
    - Edge Cases:  
      - Missing or malformed original email fields.  
      - Empty AI reply text.  
    - Sub-workflow: None.

#### 1.4 Draft Saving

- **Overview:**  
  Saves the prepared email draft into Gmail using OAuth2 credentials.

- **Nodes Involved:**  
  - Save as Gmail Draft

- **Node Details:**  

  - **Save as Gmail Draft:**  
    - Type: Gmail node  
    - Role: Creates a draft message in Gmail with the provided subject and body.  
    - Configuration:  
      - `message`: Uses the `text` field from "Prepare Email Content".  
      - `subject`: Uses the `subject` field from "Prepare Email Content".  
      - `resource`: Set to `"draft"` to save as draft instead of sending.  
      - No additional options (e.g., no attachments, no formatting).  
    - Credentials: Gmail OAuth2 account configured.  
    - Input: From "Prepare Email Content".  
    - Output: Confirmation or metadata of the draft created (not used further).  
    - Version: 2.1  
    - Edge Cases:  
      - Gmail API authentication errors.  
      - Rate limits or quota exceeded.  
      - Invalid email formatting causing draft rejection.  
    - Sub-workflow: None.

---

### 3. Summary Table

| Node Name            | Node Type                     | Functional Role                          | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                  |
|----------------------|-------------------------------|----------------------------------------|-----------------------|------------------------|----------------------------------------------------------------------------------------------|
| Sticky Note          | Sticky Note                   | Workflow annotation                     |                       |                        | This workflow triggers from Mail (IMAP), processes text with Basic LLM or Ollama Model, sets up an email field, and saves it as a draft. |
| Check New Email (IMAP)| Email Read (IMAP)             | Input reception, fetch new emails      |                       | Process Email with AI   |                                                                                              |
| Process Email with AI | LangChain LLM Chain           | AI text generation for email reply     | Check New Email (IMAP) | Prepare Email Content   |                                                                                              |
| Custom AI Model       | LangChain Ollama LM           | Alternative AI model (Ollama llama3.2) | Process Email with AI (ai_languageModel) |                        |                                                                                              |
| Prepare Email Content | Set                           | Format email draft fields               | Process Email with AI  | Save as Gmail Draft     |                                                                                              |
| Save as Gmail Draft   | Gmail node                   | Save email as draft in Gmail             | Prepare Email Content  |                        |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create IMAP Email Read Node:**  
   - Add node: **Email Read (IMAP)**  
   - Name it: `Check New Email (IMAP)`  
   - Configure with your IMAP credentials (host, port, SSL, username, password or OAuth).  
   - Leave options default or specify mailbox/folder if needed.  
   - This node triggers the workflow when a new email arrives.

2. **Add AI Processing Node (LangChain LLM Chain):**  
   - Add node: **LangChain LLM Chain**  
   - Name it: `Process Email with AI`  
   - Set parameter "text" to: `={{ $json.textPlain }}` (to extract email plaintext body)  
   - Define a prompt template with:  
     ```
     this is email

     {{ $json.textPlain }}

     i need reply of this 
     ```  
   - Select prompt type as "define".  
   - Connect the output of `Check New Email (IMAP)` to this node.

3. **Add Custom AI Model Node (LangChain Ollama):**  
   - Add node: **LangChain Ollama LM**  
   - Name it: `Custom AI Model`  
   - Configure model to `llama3.2-16000:latest`  
   - Use Ollama API credentials (API key or token).  
   - Connect this node as `ai_languageModel` output of `Process Email with AI` (optional, for alternative AI processing).

4. **Add Set Node for Email Preparation:**  
   - Add node: **Set**  
   - Name it: `Prepare Email Content`  
   - Add assignments:  
     - `from` (string): `={{ $('Check New Email (IMAP)').first().json.from }}`  
     - `subject` (string): `=Re: {{ $('Check New Email (IMAP)').first().json.subject }}`  
     - `text` (string): `={{ $json.text }}` (output from AI node)  
   - Connect output of `Process Email with AI` to this node.

5. **Add Gmail Node to Save Draft:**  
   - Add node: **Gmail**  
   - Name it: `Save as Gmail Draft`  
   - Configure Gmail OAuth2 credentials.  
   - Set parameters:  
     - `message`: `={{ $json.text }}`  
     - `subject`: `={{ $json.subject }}`  
     - `resource`: `draft` (to save as draft)  
   - Connect `Prepare Email Content` output to this node.

6. **Workflow Activation & Testing:**  
   - Save and activate the workflow.  
   - Send test emails to the monitored IMAP inbox.  
   - Verify drafts appear in Gmail drafts folder with AI-generated replies.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                  |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow triggers from Mail (IMAP), uses Basic LLM or Ollama Model, sets up email draft, saves it. | Sticky Note on workflow overview node            |
| Ollama model used: `llama3.2-16000:latest` requiring Ollama API credentials                      | Ollama AI Model node configuration               |
| Gmail OAuth2 credentials required with draft permission enabled                                 | Gmail node authentication                         |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, a workflow automation tool. This process strictly adheres to prevailing content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.