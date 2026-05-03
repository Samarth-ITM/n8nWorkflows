YouTube Video Summarizer with Gemini AI to WhatsApp, Telegram & Google Sheets

https://n8nworkflows.xyz/workflows/youtube-video-summarizer-with-gemini-ai-to-whatsapp--telegram---google-sheets-8670


# YouTube Video Summarizer with Gemini AI to WhatsApp, Telegram & Google Sheets

### 1. Workflow Overview

This workflow automates the summarization and transcription of YouTube videos using Google's Gemini AI (PaLM API). It targets content creators, marketers, educators, and anyone who needs quick, structured English summaries or full transcripts of YouTube videos up to 30 minutes in length. The workflow accepts video links via multiple input channels, processes them through the AI model, and distributes the outputs via messaging platforms and Google Sheets.

**Logical Blocks:**

- **1.1 Input Reception:** Handles three different triggers to receive YouTube video links: Form submission, WhatsApp message, and Telegram message.
- **1.2 AI Processing:** Sends the YouTube link to Google Gemini AI for generating a structured summary or full transcript in English.
- **1.3 Output Distribution:** Updates a Google Sheet with the transcript and sends the summaries/transcripts back to users via WhatsApp or Telegram messages.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for user input via three distinct channels: a web form, WhatsApp messages, and Telegram messages. Each trigger initiates the workflow by capturing the YouTube video link provided by the user.

**Nodes Involved:**  
- On form submission  
- WhatsApp Trigger (disabled)  
- Telegram Trigger (disabled)  
- Sticky Note (explains triggers)

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Primary user interface to collect YouTube video URLs via a web form titled "Youtube video summerizer."  
  - *Configuration:* Single required field labeled "YouTube Link" with placeholder text to paste the link.  
  - *Inputs:* External user submission via webhook.  
  - *Outputs:* Passes data to "Message a model" node.  
  - *Edge Cases:* Invalid or missing YouTube URLs; no validation on URL format beyond required field.  
  - *Version:* 2.3  

- **WhatsApp Trigger** (disabled)  
  - *Type:* WhatsApp Trigger  
  - *Role:* Listens to incoming WhatsApp messages to trigger workflow.  
  - *Configuration:* Monitors "messages" updates only.  
  - *Inputs:* Incoming WhatsApp chat messages via webhook.  
  - *Outputs:* Sends received message content to AI processing node.  
  - *Edge Cases:* Disabled by default; requires WhatsApp API credentials and activation.  
  - *Version:* 1  

- **Telegram Trigger** (disabled)  
  - *Type:* Telegram Trigger  
  - *Role:* Triggers workflow upon new Telegram messages.  
  - *Configuration:* Monitors "message" updates.  
  - *Inputs:* Telegram bot messages via webhook.  
  - *Outputs:* Feeds message content to "Message a model" AI node.  
  - *Credentials:* Requires Telegram API credentials.  
  - *Edge Cases:* Disabled by default; requires activation and valid credentials.  
  - *Version:* 1.2  

- **Sticky Note (Triggers Explanation)**  
  - Explains that the workflow can start by three triggers: WhatsApp, Form, Telegram.  
  - Alerts that WhatsApp and Telegram triggers are disabled by default.  

---

#### 2.2 AI Processing

**Overview:**  
This block processes the YouTube link by sending it to Google Gemini AI (PaLM API) for either a full translated transcript or a concise, structured summary in English. It enforces constraints such as video length limit (30 minutes) and language output (English only).

**Nodes Involved:**  
- Message a model  
- Sticky Note (Gemini AI details)

**Node Details:**

- **Message a model**  
  - *Type:* Google Gemini AI (PaLM API) node  
  - *Role:* Sends prompts and YouTube link to Gemini AI for summarization or transcription.  
  - *Configuration:*  
    - Model ID set to "models/gemini-2.0-flash."  
    - Messages include detailed instructions to act as a YouTube Summarizer AI with two modes:  
      - Summarize Video (structured bullet points, neutral tone)  
      - Full Transcript (translated, coherent English transcript)  
    - Input dynamically uses `{{ $json['YouTube Link'] }}` from trigger data.  
    - Enforces max video length of 30 minutes.  
    - Outputs only in English.  
  - *Inputs:* Receives YouTube link from any trigger node.  
  - *Outputs:* Passes AI response to output distribution nodes.  
  - *Credentials:* Requires Google Gemini (PaLM) API credentials.  
  - *Edge Cases:* Input video longer than 30 minutes (should not process); API auth errors; request timeouts; malformed prompt inputs.  
  - *Version:* 1  

- **Sticky Note (Gemini AI Purpose & Instructions)**  
  - Describes the purpose of Gemini AI node and summarizes key prompt instructions and constraints.  

---

#### 2.3 Output Distribution

**Overview:**  
This block handles the results from the AI node by saving the transcript to Google Sheets and sending messages back to users via WhatsApp or Telegram. The WhatsApp and Telegram output nodes are disabled by default.

**Nodes Involved:**  
- Update transcript into Sheet  
- Send message (WhatsApp, disabled)  
- Send a text message (Telegram, disabled)  
- Sticky Notes (output explanation and activation note)

**Node Details:**

- **Update transcript into Sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Updates a specific Google Sheet with the AI-generated transcript.  
  - *Configuration:*  
    - Operation: Update  
    - Sheet ID and document ID point to the target Google Sheets document and sheet (gid=0).  
    - Updates row number 2 with the "Full Transcipt" field set to the AI output.  
    - Mapping mode defines columns explicitly to match the incoming data.  
  - *Inputs:* Receives AI response from "Message a model" node.  
  - *Outputs:* None connected (end node).  
  - *Credentials:* Requires Google Sheets OAuth2 credentials.  
  - *Edge Cases:* Sheet access permission errors; invalid range or document ID; concurrency issues if multiple updates happen simultaneously.  
  - *Version:* 4.7  

- **Send message (WhatsApp)** (disabled)  
  - *Type:* WhatsApp node  
  - *Role:* Sends the AI summary or transcript back to WhatsApp users.  
  - *Configuration:*  
    - Operation: Send  
    - Placeholder recipient phone number and text body (to be replaced in production).  
  - *Inputs:* Receives AI output.  
  - *Credentials:* Requires WhatsApp API credentials.  
  - *Edge Cases:* Disabled by default; requires activation and valid phone number; potential message size limits.  
  - *Version:* 1  

- **Send a text message (Telegram)** (disabled)  
  - *Type:* Telegram node  
  - *Role:* Sends the AI output via Telegram message.  
  - *Configuration:* Uses default text body placeholder.  
  - *Inputs:* Receives AI output.  
  - *Credentials:* Requires Telegram API credentials.  
  - *Edge Cases:* Disabled by default; requires activation; message size limits; potential rate limits.  
  - *Version:* 1.2  

- **Sticky Note (Output Explanation)**  
  - Describes the three main output channels: WhatsApp message, Google Sheets, Telegram message.  
  - Explains that WhatsApp and Telegram nodes are disabled by default and need activation.  

- **Sticky Note (Activation Note)**  
  - Indicates how to activate WhatsApp and Telegram nodes by selecting and pressing 'D' on keyboard and configuring credentials.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                      | Input Node(s)         | Output Node(s)                         | Sticky Note                                                                                                                                                                                                                      |
|-----------------------|----------------------------------|------------------------------------|-----------------------|---------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| On form submission    | Form Trigger                     | Receives YouTube link via form     | (external webhook)    | Message a model                       | This workflow can be started in three ways (triggers): WhatsApp Trigger, Form Trigger, Telegram Trigger.                                                                                                                        |
| WhatsApp Trigger      | WhatsApp Trigger (disabled)      | Receives YouTube link via WhatsApp | (external webhook)    | Message a model                       | This workflow can be started in three ways (triggers): WhatsApp Trigger, Form Trigger, Telegram Trigger.                                                                                                                        |
| Telegram Trigger      | Telegram Trigger (disabled)       | Receives YouTube link via Telegram | (external webhook)    | Message a model                       | This workflow can be started in three ways (triggers): WhatsApp Trigger, Form Trigger, Telegram Trigger.                                                                                                                        |
| Message a model       | Google Gemini AI                  | Sends YouTube link for AI processing | On form submission, WhatsApp Trigger, Telegram Trigger | Update transcript into Sheet, Send message, Send a text message | Purpose: Uses Google Gemini (PaLM) API for summarization and transcription with rules and constraints. See sticky note for detailed prompt instructions.                                                                         |
| Update transcript into Sheet | Google Sheets                | Updates Google Sheet with transcript | Message a model       | (none)                              | The workflow can send results to WhatsApp, Google Sheets, Telegram.                                                                                                                                                              |
| Send message          | WhatsApp node (disabled)          | Sends summary/transcript to WhatsApp | Message a model       | (none)                              | The workflow can send results to WhatsApp, Google Sheets, Telegram. Activate by selecting node and pressing 'D'.                                                                                                               |
| Send a text message   | Telegram node (disabled)           | Sends summary/transcript to Telegram | Message a model       | (none)                              | The workflow can send results to WhatsApp, Google Sheets, Telegram. Activate by selecting node and pressing 'D'.                                                                                                               |
| Sticky Note           | Sticky Note                      | Explains triggers and start options | (none)                | (none)                              | This workflow can be started in three ways (triggers): WhatsApp Trigger, Form Trigger, Telegram Trigger.                                                                                                                        |
| Sticky Note1          | Sticky Note                      | Explains output options             | (none)                | (none)                              | The workflow can send results to WhatsApp, Google Sheets, Telegram.                                                                                                                                                              |
| Sticky Note2          | Sticky Note                      | Explains Gemini AI setup and prompt | (none)                | (none)                              | Purpose: Uses Google Gemini (PaLM API) for AI processing. See node for model ID and instructions.                                                                                                                               |
| Sticky Note3          | Sticky Note                      | Notes on activation of WhatsApp and Telegram nodes | (none)                | (none)                              | WhatsApp and Telegram nodes are deactivated by default; activate by pressing 'D' and configure credentials.                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Name: "On form submission"  
   - Configure form titled "Youtube video summerizer" with one required field:  
     - Label: "YouTube Link"  
     - Placeholder: "Paste YouTube Link"  
   - Save and note webhook URL generated.  

2. **Set Up WhatsApp Trigger Node (Optional)**  
   - Type: WhatsApp Trigger  
   - Name: "WhatsApp Trigger"  
   - Monitor updates: "messages"  
   - Connect WhatsApp API credentials (OAuth or API key).  
   - Note: This node is disabled by default; activate if needed.

3. **Set Up Telegram Trigger Node (Optional)**  
   - Type: Telegram Trigger  
   - Name: "Telegram Trigger"  
   - Monitor updates: "message"  
   - Connect Telegram API credentials (bot token).  
   - Note: This node is disabled by default; activate if needed.

4. **Create Google Gemini AI Node**  
   - Type: Google Gemini (PaLM API)  
   - Name: "Message a model"  
   - Select model ID: "models/gemini-2.0-flash"  
   - Configure messages with two-part prompt:  
     - Role "model": Instructions for YouTube Summarizer AI with modes (summary & transcript), constraints (max 30 min, English only), and input variable `{{ $json['YouTube Link'] }}`  
     - User prompt: Request full transcript for the YouTube link provided.  
   - Connect Google Gemini API credentials.  

5. **Connect Triggers to AI Node**  
   - Connect "On form submission" main output to "Message a model" input.  
   - Connect WhatsApp Trigger and Telegram Trigger (if activated) similarly to "Message a model."  

6. **Create Google Sheets Node**  
   - Type: Google Sheets  
   - Name: "Update transcript into Sheet"  
   - Operation: Update  
   - Document ID: Use your target Google Sheets document ID.  
   - Sheet name: Use sheet ID (e.g., gid=0).  
   - Define columns:  
     - "Full Transcipt" (string) — mapped to AI output transcript  
     - "row_number" (number) — set to 2 as the row to update  
   - Connect Google Sheets OAuth2 credentials.  

7. **Connect AI Node Output to Google Sheets Node**  
   - From "Message a model" output to "Update transcript into Sheet" input.

8. **Create WhatsApp Send Message Node (Optional)**  
   - Type: WhatsApp  
   - Name: "Send message"  
   - Operation: Send  
   - Configure recipient phone number (dynamic or static)  
   - Text body to contain AI output (summary or transcript).  
   - Connect WhatsApp API credentials.  
   - Note: Disabled by default; activate if used.

9. **Create Telegram Send Message Node (Optional)**  
   - Type: Telegram  
   - Name: "Send a text message"  
   - Configure to send AI output text.  
   - Connect Telegram API credentials.  
   - Note: Disabled by default; activate if used.

10. **Connect AI Node Outputs to Messaging Nodes**  
    - From "Message a model" output to "Send message" (WhatsApp) and "Send a text message" (Telegram) nodes.

11. **Add Sticky Notes for Documentation (Optional but Recommended)**  
    - Add notes explaining triggers, AI processing, outputs, and activation instructions for messaging nodes.

12. **Activation and Testing**  
    - Enable desired trigger and output nodes by selecting and pressing 'D'.  
    - Test each input channel with a valid YouTube link under 30 minutes.  
    - Verify AI responses in Google Sheets and messaging channels.

---

### 5. General Notes & Resources

| Note Content                                                                                                                  | Context or Link                                                                                           |
|-------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| WhatsApp and Telegram nodes are deactivated by default for safety; enable manually by selecting node and pressing 'D'.         | Activation instructions inside Sticky Note3 node.                                                        |
| Google Gemini AI (PaLM API) is used with model "models/gemini-2.0-flash" for advanced YouTube summarization and transcription. | Refer to Google Cloud documentation for API setup and quotas: https://cloud.google.com/vertex-ai/docs    |
| Workflow supports videos up to 30 minutes; longer videos are not processed as per AI prompt constraints.                      | Enforced in AI prompt instructions within "Message a model" node.                                        |
| Google Sheets document and sheet IDs must be valid and accessible by provided OAuth2 credentials to update transcripts.        | Set your own Google Sheets Document ID and Sheet GID in the "Update transcript into Sheet" node.          |
| Workflow design enables multi-channel input and multi-channel output for flexible user interaction and record keeping.        | See Sticky Notes explaining triggers and output options for user guidance.                                |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled are legal and publicly accessible.