Transcribe Telegram Voice Notes to Google Docs with OpenAI & DeepSeek Summaries

https://n8nworkflows.xyz/workflows/transcribe-telegram-voice-notes-to-google-docs-with-openai---deepseek-summaries-4938


# Transcribe Telegram Voice Notes to Google Docs with OpenAI & DeepSeek Summaries

---

### 1. Workflow Overview

This workflow automates the transcription of Telegram voice notes into Google Docs documents, enhanced by AI summarization. It listens for new voice notes on Telegram, uses OpenAI for transcription, applies DeepSeek-powered AI agents for summarization or advanced processing, and finally saves the results into Google Drive as Google Docs.

Logical blocks:

- **1.1 Telegram Input Reception:** Receives voice notes from Telegram via webhook trigger and Telegram node.
- **1.2 OpenAI Transcription:** Sends audio data to OpenAI for transcription.
- **1.3 AI Agent Processing with DeepSeek:** Processes the transcription through DeepSeek chat model and AI agent for summarization or further NLP tasks.
- **1.4 Google Drive Storage:** Saves the transcribed and processed content as Google Docs in Google Drive.
- **1.5 Trigger and Control Nodes:** Telegram Trigger node to initiate the workflow.
- **1.6 Informational Sticky Notes:** Provide contextual or instructional comments within the workflow.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input Reception

**Overview:**  
This block captures incoming voice notes from Telegram users. It uses a Telegram Trigger node to listen for messages, then passes data to the Telegram node for detailed processing.

**Nodes Involved:**  
- Telegram Trigger1  
- Telegram

**Node Details:**  

- **Telegram Trigger1**  
  - Type: Telegram Trigger (Webhook listener for Telegram updates)  
  - Configuration: Uses a webhook ID to receive Telegram updates (messages, voice notes) in real-time. No additional filters configured.  
  - Input: Incoming HTTP requests from Telegram servers.  
  - Output: Raw Telegram update object, including voice note metadata.  
  - Edge Cases: Telegram webhook connectivity issues, invalid updates, or unsupported message types may cause no data.  
  - Version: 1  

- **Telegram**  
  - Type: Telegram node for message processing  
  - Configuration: Default parameters; likely configured to handle voice note retrieval from messages triggered upstream.  
  - Input: Output from Telegram Trigger1.  
  - Output: Processed message content including voice note file info or media URL for transcription.  
  - Edge Cases: Failures in retrieving voice note content, expired media URLs, or Telegram API rate limits.  
  - Version: 1.2  

---

#### 2.2 OpenAI Transcription

**Overview:**  
This block sends the Telegram voice note audio data to OpenAI’s transcription service and receives the text transcription.

**Nodes Involved:**  
- OpenAI

**Node Details:**  

- **OpenAI**  
  - Type: OpenAI node (Langchain integration)  
  - Configuration: No explicit parameters shown, but assumed configured for transcription (likely using Whisper model or equivalent).  
  - Input: Voice note audio data from Telegram node.  
  - Output: Transcribed text of the voice note.  
  - Credentials: Requires OpenAI API credentials (token/key) configured in n8n.  
  - Edge Cases: API authentication errors, audio format issues, timeouts, transcription inaccuracies.  
  - Version: 1  

---

#### 2.3 AI Agent Processing with DeepSeek

**Overview:**  
After transcription, the text is forwarded to an AI Agent powered by DeepSeek’s chat model for summarization or advanced NLP processing.

**Nodes Involved:**  
- AI Agent2  
- DeepSeek Chat Model2

**Node Details:**  

- **DeepSeek Chat Model2**  
  - Type: DeepSeek Language Model Chat node  
  - Configuration: Acts as the language model backend for the AI Agent node.  
  - Input: Receives text from AI Agent2 as the language model interface.  
  - Output: Returns processed text or responses back to AI Agent2.  
  - Credentials: DeepSeek API credentials required.  
  - Edge Cases: API latency, auth failures, model capacity or rate limits.  
  - Version: 1  

- **AI Agent2**  
  - Type: Langchain AI Agent node  
  - Configuration: Uses DeepSeek Chat Model2 as its language model; likely configured with agent prompts and task instructions for summarization or note enhancement.  
  - Input: Receives transcribed text from OpenAI node.  
  - Output: Summarized or enhanced text sent to Google Drive node.  
  - Edge Cases: Misconfiguration of agent prompts leading to irrelevant output, API failures, or unexpected input formats.  
  - Version: 1.9  

---

#### 2.4 Google Drive Storage

**Overview:**  
This block saves the final AI-processed transcription and summaries into Google Drive as Google Docs documents.

**Nodes Involved:**  
- Google Drive1  
- Google Drive3

**Node Details:**  

- **Google Drive1**  
  - Type: Google Drive node  
  - Configuration: Likely configured for file creation or uploading the OpenAI transcription output.  
  - Input: Receives text from OpenAI and AI Agent2 nodes.  
  - Output: File metadata or confirmation, forwarded to AI Agent2 for further processing.  
  - Credentials: Google OAuth2 credentials configured.  
  - Edge Cases: Drive quota exceeded, authentication expiration, file permission issues.  
  - Version: 3  

- **Google Drive3**  
  - Type: Google Drive node  
  - Configuration: Receives final output from AI Agent2, stores final Google Docs file in Drive.  
  - Input: Summarized or enhanced text from AI Agent2.  
  - Output: File metadata confirmation as last step.  
  - Credentials: Same as Google Drive1.  
  - Edge Cases: Same quota, auth, or permission risks.  
  - Version: 3  

---

#### 2.5 Sticky Notes (Contextual Comments)

**Overview:**  
Sticky notes are included throughout the workflow presumably for annotations or instructions.

**Nodes Involved:**  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note  

**Node Details:**  

- All sticky notes have empty content fields in the provided JSON.  
- They serve a purely visual or organizational purpose; no technical configuration or connections.  

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                     | Input Node(s)      | Output Node(s)          | Sticky Note |
|---------------------|----------------------------------|-----------------------------------|--------------------|-------------------------|-------------|
| Telegram Trigger1    | Telegram Trigger                 | Listen for Telegram messages      | -                  | Telegram                |             |
| Telegram            | Telegram                        | Retrieve voice note media          | Telegram Trigger1  | OpenAI                  |             |
| OpenAI              | Langchain OpenAI                | Transcribe voice note audio        | Telegram            | Google Drive1, AI Agent2|             |
| Google Drive1       | Google Drive                    | Save transcription intermediate    | OpenAI              | AI Agent2               |             |
| AI Agent2           | Langchain AI Agent              | Summarize/process transcription    | OpenAI, Google Drive1| Google Drive3           |             |
| DeepSeek Chat Model2| DeepSeek Language Model Chat    | Backend language model for AI Agent| AI Agent2 (ai_languageModel) | AI Agent2 (ai_languageModel) |             |
| Google Drive3       | Google Drive                    | Save final summarized document     | AI Agent2            | -                       |             |
| Sticky Note1        | Sticky Note                    | Annotation/Comment                  | -                  | -                       |             |
| Sticky Note2        | Sticky Note                    | Annotation/Comment                  | -                  | -                       |             |
| Sticky Note3        | Sticky Note                    | Annotation/Comment                  | -                  | -                       |             |
| Sticky Note4        | Sticky Note                    | Annotation/Comment                  | -                  | -                       |             |
| Sticky Note         | Sticky Note                    | Annotation/Comment                  | -                  | -                       |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Node Type: Telegram Trigger  
   - Configure with your Telegram Bot token and set up webhook URL in Telegram Bot API.  
   - No additional filters; listens for all messages.  
   - Position: Starting point.

2. **Create Telegram Node**  
   - Node Type: Telegram  
   - Connect input from Telegram Trigger node.  
   - Configure to retrieve voice note media (default settings usually suffice if connected properly).  
   - Ensure webhookId matches or is set automatically.

3. **Create OpenAI Node**  
   - Node Type: Langchain OpenAI  
   - Connect input from Telegram node.  
   - Configure for audio transcription (select Whisper or equivalent model).  
   - Add OpenAI API credentials (API key).  
   - Set appropriate parameters for transcription (language, audio format if required).

4. **Create Google Drive Node (Google Drive1)**  
   - Node Type: Google Drive  
   - Connect input from OpenAI node.  
   - Configure for file creation/upload with Google Docs MIME type.  
   - Add OAuth2 Google Drive credentials.  
   - Set folder path or parent folder ID where transcriptions will be saved.

5. **Create AI Agent Node (AI Agent2)**  
   - Node Type: Langchain AI Agent  
   - Connect input from OpenAI node and Google Drive1 node (as per workflow connections).  
   - Configure agent with appropriate prompt or instructions for summarization.  
   - Set DeepSeek Chat Model2 as the language model for this agent.  
   - Provide DeepSeek API credentials.

6. **Create DeepSeek Chat Model Node (DeepSeek Chat Model2)**  
   - Node Type: DeepSeek Language Model Chat  
   - Connect as language model backend to AI Agent2 node’s `ai_languageModel` input.  
   - Configure API credentials and model parameters.

7. **Create Google Drive Node (Google Drive3)**  
   - Node Type: Google Drive  
   - Connect input from AI Agent2 node.  
   - Configure to save the final summarized document as Google Doc.  
   - Use same Google Drive credentials as Google Drive1.  
   - Specify folder location or naming conventions.

8. **Optional: Create Sticky Notes**  
   - Add sticky notes to document workflow parts as needed for team collaboration or reminders.

9. **Test End-to-End**  
   - Send a voice note to your Telegram bot.  
   - Monitor each node execution for errors or data flow.  
   - Adjust configurations or API limits as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                       |
|----------------------------------------------------------------------------------------------|-------------------------------------------------------|
| Use Telegram Bot API documentation to correctly set up webhooks and bot permissions.        | https://core.telegram.org/bots/api                   |
| OpenAI Whisper API or relevant endpoint for audio transcription is recommended.             | https://platform.openai.com/docs/models/speech-to-text|
| DeepSeek AI integration requires API credentials and understanding of agent prompt design. | https://deepseek.ai/docs                              |
| Google Drive OAuth2 credentials must include Drive API scopes for file creation/editing.    | https://developers.google.com/drive/api/v3/about-auth|
| Ensure all API quotas and rate limits are respected to avoid workflow failures.              | See each API provider’s documentation                  |

---

**Disclaimer:**  
The provided text is generated from an automated workflow designed with n8n, adhering strictly to current content policies. It contains no illegal, offensive, or protected material. All processed data is legal and public.

---