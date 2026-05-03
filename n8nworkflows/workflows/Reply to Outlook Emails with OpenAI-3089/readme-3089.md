Reply to Outlook Emails with OpenAI

https://n8nworkflows.xyz/workflows/reply-to-outlook-emails-with-openai-3089


# Reply to Outlook Emails with OpenAI

### 1. Workflow Overview

This workflow automates replying to Microsoft Outlook emails using a trained AI agent powered by OpenAI. It is designed for Outlook users who want an AI to emulate their personal or shared inbox writing style and tone when responding to selected senders. The workflow includes logical blocks for email reception and filtering, AI model configuration and instruction, and automated reply generation with options for direct sending or saving drafts.

Logical blocks:

- **1.1 Input Reception and Filtering:** Connects to Outlook, triggers on incoming emails, and filters by sender.
- **1.2 AI Processing and Instruction Setup:** Configures the OpenAI chat model and defines AI agent instructions including persona and style.
- **1.3 Reply Generation and Sending:** Uses AI-generated replies to respond to emails via Outlook, with options to send immediately or save as draft.
- **1.4 Documentation and Guidance:** Sticky notes provide user instructions and tips throughout the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

**Overview:**  
This block connects to the user's Outlook account and triggers the workflow when an email is received from specified sender(s). It filters incoming emails to only those from selected addresses to focus AI responses.

**Nodes Involved:**  
- Connect Outlook & Set Filter  
- Sticky Note (Trigger Action)

**Node Details:**

- **Connect Outlook & Set Filter**  
  - Type: Microsoft Outlook Trigger  
  - Role: Listens for new incoming emails in Outlook, filtering by sender email address.  
  - Configuration:  
    - Output set to "raw" to pass full email data.  
    - Filter configured to trigger only on emails from "sales@yourcompany.com" (example).  
    - Polling interval set to every minute for near real-time detection.  
  - Inputs: None (trigger node)  
  - Outputs: Passes filtered email data to AI instruction node.  
  - Credentials: Requires Microsoft Outlook OAuth2 authentication.  
  - Edge Cases:  
    - Authentication failure or token expiration.  
    - No emails matching filter may cause no triggers.  
    - Polling delay may affect responsiveness.  
  - Sticky Note: "Trigger Action" explains connection and filter setup.

- **Sticky Note (Trigger Action)**  
  - Provides instructions on connecting Outlook, setting trigger to "message received," output to "raw," and filtering sender addresses.

---

#### 2.2 AI Processing and Instruction Setup

**Overview:**  
This block configures the OpenAI chat model and sets detailed AI agent instructions to emulate the user's writing style and tone when replying to emails.

**Nodes Involved:**  
- Add OpenAI Chat Model  
- Add AI Agent Instructions  
- Sticky Note1 (Agent Instructions)  
- Sticky Note2 (Add AI Model)  
- Sticky Note3 (Reply Settings)  
- Sticky Note4 (Draft a Reply)

**Node Details:**

- **Add OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides AI language model interface for generating replies.  
  - Configuration:  
    - Model selected: "gpt-4o-mini" (example).  
    - No additional options configured.  
  - Inputs: Receives data from previous node (email trigger).  
  - Outputs: Passes model interface to AI agent instructions node.  
  - Credentials: Requires OpenAI API key.  
  - Edge Cases:  
    - API key invalid or rate limits exceeded.  
    - Model unavailability or version mismatch.  
  - Sticky Note: "Add AI Model" explains credential and model selection.

- **Add AI Agent Instructions**  
  - Type: LangChain Agent Node  
  - Role: Defines the prompt and system message guiding AI reply generation.  
  - Configuration:  
    - Prompt includes email metadata (ID, sender, subject, message body) formatted in XML-like tags.  
    - System message defines AI role, capabilities, limitations, and response style.  
    - User is instructed to replace placeholders with their name and paste sample email replies between `<example>` tags to train the agent's tone.  
    - Prompt type set to "define" for custom instruction.  
  - Inputs: Receives email data and AI model from previous nodes.  
  - Outputs: Provides AI-generated reply text.  
  - Edge Cases:  
    - Expression errors if JSON fields missing or malformed.  
    - Insufficient or inconsistent sample replies may degrade AI output quality.  
  - Sticky Note: "Agent Instructions" details prompt structure and training examples.

- **Sticky Note1 (Agent Instructions)**  
  - Contains detailed instructions on AI assistant role, limitations, and examples for tone training.

- **Sticky Note3 (Reply Settings)**  
  - Explains manual setting of reply parameters, toggling reply to sender only, and letting AI define message content.

- **Sticky Note4 (Draft a Reply)**  
  - Describes option to save AI-generated replies as drafts for human review before sending.

---

#### 2.3 Reply Generation and Sending

**Overview:**  
This block sends the AI-generated reply back through Outlook, replying to the original email thread. It supports replying only to the sender and setting reply metadata.

**Nodes Involved:**  
- Reply to Email  
- Sticky Note3 (Reply Settings)  
- Sticky Note4 (Draft a Reply)  

**Node Details:**

- **Reply to Email**  
  - Type: Microsoft Outlook Tool  
  - Role: Sends a reply email using AI-generated content.  
  - Configuration:  
    - Operation: "reply" to existing email thread.  
    - Message content dynamically set from AI agent output via expression.  
    - Message ID set to original email's ID to maintain thread.  
    - Reply only to sender enabled.  
    - Additional fields set reply-to address and subject from original email.  
  - Inputs: Receives AI-generated message and original email metadata.  
  - Outputs: None (final node).  
  - Credentials: Requires Microsoft Outlook OAuth2 authentication.  
  - Edge Cases:  
    - Failure to send due to network or auth errors.  
    - Message content empty or invalid.  
    - Reply threading may fail if message ID is incorrect.  
  - Sticky Notes: "Reply Settings" and "Draft a Reply" provide configuration guidance.

---

#### 2.4 Documentation and Guidance

**Overview:**  
Sticky notes throughout the workflow provide user guidance, tips, and instructions for setup and usage.

**Nodes Involved:**  
- Sticky Note (Trigger Action)  
- Sticky Note1 (Agent Instructions)  
- Sticky Note2 (Add AI Model)  
- Sticky Note3 (Reply Settings)  
- Sticky Note4 (Draft a Reply)

**Node Details:**  
Sticky notes contain detailed instructions on connecting accounts, setting filters, defining AI instructions, selecting models, configuring reply behavior, and draft options. They include tips for training the AI with sample replies and toggling draft mode.

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                          | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                   |
|---------------------------|--------------------------------------|----------------------------------------|-----------------------------|----------------------------|---------------------------------------------------------------------------------------------------------------|
| Connect Outlook & Set Filter | Microsoft Outlook Trigger             | Triggers workflow on filtered incoming emails | None                        | Add AI Agent Instructions   | Trigger Action: Connect Outlook, set trigger to message received, output raw, filter sender addresses          |
| Add OpenAI Chat Model      | LangChain OpenAI Chat Model           | Provides AI language model interface   | None (triggered by email)   | Add AI Agent Instructions   | Add AI Model: OpenAI credentials required, select model (e.g., gpt-4o-mini)                                    |
| Add AI Agent Instructions  | LangChain Agent                      | Defines AI prompt and instructions     | Connect Outlook & Set Filter, Add OpenAI Chat Model | Reply to Email             | Agent Instructions: Define AI role, capabilities, response style, and provide sample reply examples            |
| Reply to Email             | Microsoft Outlook Tool                | Sends AI-generated reply email         | Add AI Agent Instructions   | None                       | Reply Settings: Set reply operation, reply only to sender, dynamic message; Draft a Reply: option to save draft |
| Sticky Note                | Sticky Note                         | Provides user instructions              | None                        | None                       | Trigger Action instructions                                                                                     |
| Sticky Note1               | Sticky Note                         | Provides AI agent instruction guidance  | None                        | None                       | Agent Instructions content                                                                                        |
| Sticky Note2               | Sticky Note                         | Provides AI model setup guidance        | None                        | None                       | Add AI Model instructions                                                                                        |
| Sticky Note3               | Sticky Note                         | Provides reply configuration guidance  | None                        | None                       | Reply Settings instructions                                                                                      |
| Sticky Note4               | Sticky Note                         | Provides draft reply option instructions | None                        | None                       | Draft a Reply instructions                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node: Connect Outlook & Set Filter**  
   - Add a **Microsoft Outlook Trigger** node.  
   - Set **Output** to "raw" to receive full email data.  
   - Under **Filters**, add a filter for **Sender** with the email address(es) you want the AI to respond to (e.g., "sales@yourcompany.com").  
   - Set **Poll Times** to "every minute" for near real-time triggering.  
   - Configure **Microsoft Outlook OAuth2** credentials with your Outlook account.

2. **Add OpenAI Chat Model Node**  
   - Add a **LangChain OpenAI Chat Model** node.  
   - Select your OpenAI credentials (API key).  
   - Choose the AI model (e.g., "gpt-4o-mini").  
   - Leave additional options default unless customization is needed.

3. **Add AI Agent Instructions Node**  
   - Add a **LangChain Agent** node.  
   - Set the **Prompt** text to:  
     ```
     Write a reply to the following email, then save it as a draft to the email thread:
     <email>
     ID: {{ $json.id }}
     From: {{ $json.from.emailAddress.address }}
     Subject: {{ $json.subject }}
     Message: {{ $json.body.content }}
     </email>
     ```  
   - Under **Options**, set the **System Message** to:  
     ```
     #role
     You are an AI assistant specializing in replying to incoming emails to [YOUR NAME] Outlook inbox.

     #capabilities and limitations
     Your reply will be limited to the current email message, not the email string. Do not hallucinate.

     #response
     Reply in a casual, modern, professional, concise writing style. You should sound like [YOUR NAME HERE]. Here are examples of [YOUR NAME HERE] voice:
     <example>
     [COPY & PASTE REPLY SAMPLES FROM YOUR EMAIL]
     </example>
     <example>
     [COPY & PASTE REPLY SAMPLES FROM YOUR EMAIL]
     </example>
     <example>
     [ADD A VARIETY OF REPLY SAMPLES SO THE AGENT UNDERSTANDS YOUR TONE & STYLE]
     </example>
     ```  
   - Set **Prompt Type** to "define".  
   - Connect inputs from both the Outlook Trigger and OpenAI Chat Model nodes.

4. **Add Reply to Email Node**  
   - Add a **Microsoft Outlook Tool** node.  
   - Set **Operation** to "reply".  
   - For **Message**, use the expression:  
     ```
     ={{ $fromAI('Message', ``, 'string') }}
     ```  
   - Set **Message ID** to the original email's ID:  
     ```
     ={{ $json.id }}
     ```  
   - Enable **Reply to Sender Only** toggle.  
   - Under **Additional Fields**, set **Reply To** to:  
     ```
     ={{ $json.sender.emailAddress.address }}
     ```  
   - Set **Subject** to:  
     ```
     ={{ $json.subject }}
     ```  
   - Configure Microsoft Outlook OAuth2 credentials.

5. **Connect Nodes**  
   - Connect **Connect Outlook & Set Filter** node output to **Add AI Agent Instructions** node input.  
   - Connect **Add OpenAI Chat Model** node output to **Add AI Agent Instructions** node input.  
   - Connect **Add AI Agent Instructions** node output to **Reply to Email** node input.

6. **Optional: Add Draft Reply Toggle**  
   - To enable saving replies as drafts instead of sending immediately, add a boolean field toggle in the **Reply to Email** node parameters (using "Add Field" option).  
   - When toggled on, configure the node to save replies in the Drafts folder instead of sending.

7. **Test Workflow**  
   - Send an email from the filtered sender address.  
   - Verify the AI-generated reply appears in Sent or Drafts folder as configured.

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                  |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| This template is intended for Microsoft Outlook users who want AI to emulate their writing style in replies. | Workflow description and target audience         |
| To train the AI agent, paste actual email replies from your sent folder between `<example>` tags in the system message. | AI agent instruction setup                        |
| Toggle draft mode to review AI replies before sending automatically.                                        | Draft reply option instructions                   |
| For support or questions, contact: support@teambisonandbird.com                                           | Support contact                                   |
| This workflow does not extract email addresses from email bodies; it only replies to filtered senders.    | Limitation notice                                 |
| Screenshots illustrating setup steps are included in the original workflow documentation (not embedded here). | Visual setup guidance                             |

---

This structured documentation provides a comprehensive understanding of the workflow’s design, node configurations, and operational logic, enabling advanced users and AI agents to reproduce, modify, and troubleshoot the workflow effectively.