Automatic Travel Itinerary Generation via Email with Llama AI

https://n8nworkflows.xyz/workflows/automatic-travel-itinerary-generation-via-email-with-llama-ai-4917


# Automatic Travel Itinerary Generation via Email with Llama AI

### 1. Workflow Overview

This workflow automates the generation of personalized travel itineraries by processing incoming emails with itinerary requests and replying with AI-generated travel plans. It is designed for travel agencies or services that want to automate customer interaction and itinerary creation using natural language processing powered by Llama AI models.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Watches an email inbox for new messages with itinerary requests.
- **1.2 AI Processing:** Uses an AI language model to generate a human-friendly travel itinerary based on the email content.
- **1.3 Output Preparation:** Sets up email fields for the response email.
- **1.4 Email Sending:** Sends the generated itinerary back to the requester via email.

Supporting these blocks are configuration notes in sticky notes guiding credential setup.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Monitors an IMAP email inbox for unseen emails whose subject contains the word “itinerary”. This triggers the workflow to process the email content.

- **Nodes Involved:**  
  - Trigger From Mail (IMAP)

- **Node Details:**

  - **Trigger From Mail (IMAP)**  
    - Type: Email Read (IMAP Trigger)  
    - Role: Watches incoming emails matching specific criteria.  
    - Configuration:  
      - Filters for unseen emails with subject containing “itinerary”.  
      - Uses IMAP credentials (configured outside the workflow).  
    - Input: None (trigger node).  
    - Output: Emits raw email data including sender, subject, and body.  
    - Edge Cases:  
      - Authentication failure if IMAP credentials are invalid or expired.  
      - Network timeouts or IMAP server unavailability.  
      - Misformatted emails or unsupported encodings can cause parsing issues.  
    - Notes: Sticky note provides detailed IMAP setup instructions.

---

#### 2.2 AI Processing

- **Overview:**  
  Processes the email content through two sequential AI nodes: first the “Basic LLM” node generates a structured itinerary text based on the email body, then the “Ollama Model” node is configured but not connected actively (appears unused or reserved for advanced AI processing).

- **Nodes Involved:**  
  - Basic LLM  
  - Ollama Model

- **Node Details:**

  - **Basic LLM**  
    - Type: LangChain LLM Chain  
    - Role: Uses an OpenAI-compatible or similar language model to create a travel itinerary text from the raw email text.  
    - Configuration:  
      - Input text is bound to the plain text content of the email (`{{$json.textPlain}}`).  
      - The prompt instructs the AI to generate a friendly, human-style travel itinerary with a specific structure: greeting, summary, daily breakdowns matching the number of days specified, key activities, hotel, transport, and a conversational tone without markdown or emojis.  
      - Includes an extensive multi-line prompt template with examples.  
    - Input: Receives email content from the IMAP trigger.  
    - Output: Produces a formatted itinerary text.  
    - Edge Cases:  
      - AI response may be truncated or incomplete if input text is too long or model times out.  
      - If the email content does not specify days, the AI may generate incorrect day counts.  
      - Model errors or quota exhaustion could cause failures.  
    - Version Requirements: Requires LangChain node version 1.6 or higher.  

  - **Ollama Model**  
    - Type: LLM Ollama Node (LangChain)  
    - Role: Configured to use a specific Llama 3.2 model, presumably for advanced or alternative AI processing.  
    - Configuration:  
      - Model set to “llama3.2-16000:latest”.  
      - Credentials for Ollama API are configured.  
    - Input/Output Connections: Outputs to Basic LLM node, but this connection is reversed in the workflow JSON (Ollama outputs to Basic LLM’s ai_languageModel input). However, the Basic LLM node is also triggered directly from the IMAP node, so Ollama Model appears unused or redundant in the current flow.  
    - Edge Cases:  
      - API authentication failures.  
      - Model availability or version mismatches.  
      - Network timeouts.  
    - Notes: This node is present but not actively integrated in the main workflow path.

---

#### 2.3 Output Preparation

- **Overview:**  
  Constructs the email fields for the outgoing message, formatting the “from”, “subject”, and “text” fields by extracting and transforming data from previous nodes.

- **Nodes Involved:**  
  - Email Field Setup

- **Node Details:**

  - **Email Field Setup**  
    - Type: Set Node  
    - Role: Prepares structured output fields for sending the response email.  
    - Configuration:  
      - `from` is set to the sender’s email from the original incoming email.  
      - `subject` is prefixed with “Re: ” followed by the original email subject.  
      - `text` is set to the generated itinerary text from the Basic LLM node output.  
    - Input: Receives the output from Basic LLM node (which has the AI-generated text).  
    - Output: Passes structured fields downstream to the email sending node.  
    - Edge Cases:  
      - Missing or malformed email addresses could cause sending failures.  
      - If AI text generation failed, `text` could be empty.

---

#### 2.4 Email Sending

- **Overview:**  
  Sends the AI-generated itinerary back to the original email sender using SMTP.

- **Nodes Involved:**  
  - Sending Email

- **Node Details:**

  - **Sending Email**  
    - Type: Email Send (SMTP)  
    - Role: Sends a plain text email reply with the itinerary.  
    - Configuration:  
      - Uses the `toEmail` field from the “from” address in the setup node.  
      - Sets the subject as prepared in the setup node.  
      - Email body is plain text from the AI-generated itinerary.  
      - Uses configured SMTP credentials (e.g., Gmail SMTP with app password).  
      - From email is fixed as XYZ@gmail.com (should be replaced with sender’s address).  
    - Input: Receives email fields from the Email Field Setup node.  
    - Output: Sends email, no further output.  
    - Edge Cases:  
      - SMTP authentication failures or incorrect credentials.  
      - Email rejected due to spam filters or invalid addresses.  
      - Network issues causing send failures.  
    - Notes: Sticky note provides SMTP setup instructions.

---

#### 2.5 Sticky Notes (Configuration Guidance)

- **Sticky Note (IMAP Setup)**  
  - Provides detailed example configuration for IMAP credentials to be used in the “Trigger From Mail (IMAP)” node, including host, port, security, username, and password (app password recommended for Gmail).

- **Sticky Note1 (SMTP Setup)**  
  - Provides detailed example configuration for SMTP credentials used in the “Sending Email” node, including user, password, host, port, and SSL/TLS options.

---

### 3. Summary Table

| Node Name           | Node Type                      | Functional Role                        | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                      |
|---------------------|--------------------------------|-------------------------------------|------------------------|-------------------------|-------------------------------------------------------------------------------------------------|
| Trigger From Mail (IMAP) | Email Read (IMAP Trigger)      | Watches inbox for itinerary emails  | None                   | Basic LLM               | Configure Mail Credentials for Trigger Mail Node: IMAP setup example provided                   |
| Ollama Model        | LangChain LLM Ollama Model     | Alternative AI model (not actively used) | None (standalone)       | Basic LLM (ai_languageModel) |                                                                                                 |
| Basic LLM           | LangChain LLM Chain            | Generates travel itinerary text     | Trigger From Mail (IMAP) / Ollama Model (ai_languageModel) | Email Field Setup       |                                                                                                 |
| Email Field Setup   | Set Node                      | Prepares email fields for reply     | Basic LLM               | Sending Email           |                                                                                                 |
| Sending Email       | Email Send (SMTP)              | Sends itinerary email to requester  | Email Field Setup       | None                    | Configure SMTP Settings for the Send Mail Node: SMTP setup example provided                     |
| Sticky Note         | Sticky Note                   | IMAP credential setup guidance      | None                   | None                    | Configure Mail Credentials for Trigger Mail Node: IMAP setup example provided                   |
| Sticky Note1        | Sticky Note                   | SMTP credential setup guidance      | None                   | None                    | Configure SMTP Settings for the Send Mail Node: SMTP setup example provided                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create IMAP Trigger Node:**  
   - Add node: “Trigger From Mail (IMAP)”  
   - Set options to filter for unseen emails with subject containing “itinerary”: `["UNSEEN", ["SUBJECT", "itinerary"]]`  
   - Configure IMAP credentials: host (e.g., imap.gmail.com), port (993), secure (true), user email, and app password.  
   - Set node to trigger on new matching emails.

2. **Create Basic LLM Node:**  
   - Add node: “Basic LLM” (LangChain LLM Chain node)  
   - Set input text expression to `{{$json.textPlain}}` from the IMAP trigger node.  
   - Configure prompt as follows (paste the detailed prompt):  
     - Ask AI to create a friendly travel itinerary based on email content, with greeting, summary, daily breakdowns matching number of days, key activities, hotel info, and transport details.  
     - Ensure prompt forbids markdown, emojis, or special characters.  
   - No custom credentials needed unless using specific API (LangChain config applies).  
   - Connect “Trigger From Mail (IMAP)” node output to this node’s input.

3. **Create Email Field Setup Node:**  
   - Add node: “Set”  
   - Assign fields:  
     - `from` = expression: `{{$node["Trigger From Mail (IMAP)"].json["from"]}}`  
     - `subject` = expression: `"Re: " + $node["Trigger From Mail (IMAP)"].json["subject"]`  
     - `text` = expression: `{{$node["Basic LLM"].json["text"]}}` (the generated itinerary)  
   - Connect “Basic LLM” output to this node input.

4. **Create Sending Email Node:**  
   - Add node: “Sending Email” (SMTP)  
   - Set parameters:  
     - To Email: `{{$json["from"]}}` (from the previous node)  
     - Subject: `{{$json["subject"]}}`  
     - Body Text: `{{$json["text"]}}`  
     - From Email: your verified sender address (e.g., your company email)  
     - Email format: Plain Text  
   - Configure SMTP credentials: host (smtp.gmail.com), port (465), SSL enabled, user email, and app password.  
   - Connect “Email Field Setup” output to this node input.

5. **(Optional) Add Ollama Model Node:**  
   - Add node: “Ollama Model” (LangChain)  
   - Select model “llama3.2-16000:latest”  
   - Configure Ollama API credentials.  
   - This node is optional and can be connected before Basic LLM for advanced AI processing if desired.

6. **Add Sticky Notes:**  
   - Add sticky notes for IMAP and SMTP credential setup instructions referencing the examples provided.

7. **Activate Workflow:**  
   - Ensure all credentials are valid.  
   - Test by sending an email with subject containing “itinerary” and a travel request.  
   - Check the reply email for the generated itinerary.

---

### 5. General Notes & Resources

| Note Content                                                                                                                             | Context or Link                                      |
|------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Use Gmail App Passwords for IMAP and SMTP authentication to avoid two-factor authentication issues.                                     | IMAP and SMTP credential setup                       |
| The AI prompt enforces a strict textual structure to ensure the itinerary matches the number of days requested in the email content.     | Prompt details inside Basic LLM node                 |
| OpenAI or Ollama models require valid API credentials and sufficient quota for seamless operation.                                       | AI nodes credential management                        |
| Email “From” address should be a verified and authorized email account to avoid spam filtering or delivery failures.                    | Sending Email node configuration                      |

---

**Disclaimer:** The provided text comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.