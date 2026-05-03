Transcribe & Summarize Telegram Voice Notes with OpenAI and DeepSeek Chat to Google Docs

https://n8nworkflows.xyz/workflows/transcribe---summarize-telegram-voice-notes-with-openai-and-deepseek-chat-to-google-docs-4944


# Transcribe & Summarize Telegram Voice Notes with OpenAI and DeepSeek Chat to Google Docs

### 1. Workflow Overview

This workflow automates the process of transcribing and summarizing voice notes received via Telegram and saving the results into Google Docs. It is designed for users who want to efficiently handle Telegram voice messages by leveraging AI-powered transcription and summarization, then archiving the output in Google Drive documents.

The workflow can be logically divided into the following blocks:

- **1.1 Input Reception**: Captures incoming voice notes from Telegram.
- **1.2 AI Processing**: Transcribes and summarizes voice notes using OpenAI and DeepSeek Chat models combined with an AI agent.
- **1.3 Output Storage**: Saves the processed transcription and summaries as Google Docs files in Google Drive.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for incoming Telegram voice messages via a Telegram webhook trigger and fetches the message details for further processing.

- **Nodes Involved:**  
  - Telegram Trigger1  
  - Telegram1

- **Node Details:**

  - **Telegram Trigger1**  
    - *Type & Role:* Telegram Trigger node; entry point that listens for new Telegram updates (voice messages).  
    - *Configuration:* Uses a webhook ID to receive updates from Telegram bot. No additional parameters set.  
    - *Input/Output:* No input; outputs new Telegram update data to Telegram1.  
    - *Failure Cases:* Possible webhook misconfiguration, Telegram API connectivity issues, or permissions errors if the bot lacks access.  
    - *Version:* 1  
    - *Sub-workflow:* None.

  - **Telegram1**  
    - *Type & Role:* Telegram node; fetches detailed message data from the Telegram update received.  
    - *Configuration:* Default settings, connected directly to Telegram Trigger1.  
    - *Input/Output:* Input from Telegram Trigger1; outputs voice note data for AI Processing.  
    - *Failure Cases:* Telegram API rate limits or malformed message data.  
    - *Version:* 1.2  
    - *Sub-workflow:* None.

#### 1.2 AI Processing

- **Overview:**  
  Transcribes the Telegram voice note with OpenAI, then uses DeepSeek Chat and an AI Agent to summarize or enrich the transcription. The agents collaborate to refine the text before saving.

- **Nodes Involved:**  
  - OpenAI2  
  - AI Agent1  
  - DeepSeek Chat Model1

- **Node Details:**

  - **OpenAI2**  
    - *Type & Role:* OpenAI node via LangChain integration; responsible for transcribing the voice note using OpenAI’s speech-to-text capabilities.  
    - *Configuration:* Uses OpenAI credentials set in n8n; no explicit parameters shown but typically configured to invoke whisper or similar models.  
    - *Input/Output:* Receives voice note data from Telegram1; outputs raw transcription to AI Agent1 and Google Drive.  
    - *Failure Cases:* API quota exceeded, invalid credentials, or audio file format unsupported.  
    - *Version:* 1  
    - *Sub-workflow:* None.

  - **AI Agent1**  
    - *Type & Role:* LangChain Agent node; orchestrates the use of language models to summarize and refine transcription.  
    - *Configuration:* Uses DeepSeek Chat Model1 as language model backend; likely configured with prompt templates to produce summaries.  
    - *Input/Output:* Input from OpenAI2; output sent to Google Drive2.  
    - *Failure Cases:* Model invocation failures, prompt errors, or timeout.  
    - *Version:* 1.9  
    - *Sub-workflow:* None.

  - **DeepSeek Chat Model1**  
    - *Type & Role:* LangChain language model node; provides conversational AI capabilities to AI Agent1.  
    - *Configuration:* Connected as ai_languageModel input to AI Agent1; likely configured with DeepSeek credentials.  
    - *Input/Output:* Receives prompt from AI Agent1; outputs language model responses back to AI Agent1.  
    - *Failure Cases:* Authentication errors, service downtime, or networking issues.  
    - *Version:* 1  
    - *Sub-workflow:* None.

#### 1.3 Output Storage

- **Overview:**  
  Stores the transcription and summary results into Google Drive as Google Docs files for archiving and future reference.

- **Nodes Involved:**  
  - Google Drive  
  - Google Drive2

- **Node Details:**

  - **Google Drive**  
    - *Type & Role:* Google Drive node; saves the raw transcription output from OpenAI2 into Drive.  
    - *Configuration:* Uses Google OAuth2 credentials; parameters likely specify folder and file metadata.  
    - *Input/Output:* Input from OpenAI2; output is file creation confirmation (not further used).  
    - *Failure Cases:* Authentication expiration, insufficient permissions, or quota limits.  
    - *Version:* 3  
    - *Sub-workflow:* None.

  - **Google Drive2**  
    - *Type & Role:* Google Drive node; saves the final summary output from AI Agent1 into Drive.  
    - *Configuration:* Uses Google OAuth2 credentials; configured to create or update Google Docs.  
    - *Input/Output:* Input from AI Agent1; output is file creation confirmation.  
    - *Failure Cases:* Same as above—auth, permission issues, or API limits.  
    - *Version:* 3  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name          | Node Type                           | Functional Role                    | Input Node(s)     | Output Node(s)          | Sticky Note |
|--------------------|-----------------------------------|----------------------------------|-------------------|-------------------------|-------------|
| Telegram Trigger1   | Telegram Trigger                  | Receives Telegram voice updates  | —                 | Telegram1               |             |
| Telegram1          | Telegram                         | Fetches Telegram message data    | Telegram Trigger1 | OpenAI2                 |             |
| OpenAI2            | OpenAI (LangChain)               | Transcribes voice note           | Telegram1         | AI Agent1, Google Drive |             |
| AI Agent1          | LangChain Agent                  | Summarizes and refines transcription | OpenAI2          | Google Drive2           |             |
| DeepSeek Chat Model1| LangChain Language Model         | Provides chat model for AI Agent | AI Agent1 (ai_languageModel) | AI Agent1          |             |
| Google Drive       | Google Drive                     | Saves transcription to Drive     | OpenAI2           | —                       |             |
| Google Drive2      | Google Drive                     | Saves summary to Drive           | AI Agent1         | —                       |             |
| Sticky Note        | Sticky Note                     | —                                | —                 | —                       |             |
| Sticky Note3       | Sticky Note                     | —                                | —                 | —                       |             |
| Sticky Note4       | Sticky Note                     | —                                | —                 | —                       |             |
| Sticky Note5       | Sticky Note                     | —                                | —                 | —                       |             |

*Note: Sticky Notes in this workflow have empty content and do not add comments.*

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Set up webhook with Telegram bot token.  
   - No additional parameters needed.  
   - This node will receive incoming Telegram voice notes.

2. **Add Telegram node**  
   - Type: Telegram  
   - Connect input from Telegram Trigger.  
   - Default configuration to fetch message data (voice note file).  
   - Ensure Telegram credentials (bot token) are configured.

3. **Add OpenAI node (LangChain)**  
   - Type: OpenAI (LangChain node)  
   - Connect input from Telegram node.  
   - Configure OpenAI credentials with a valid API key.  
   - Configure for speech-to-text transcription (e.g., Whisper model).  
   - Output raw transcription text.

4. **Add AI Agent node (LangChain)**  
   - Type: LangChain Agent  
   - Connect input from OpenAI node.  
   - Configure the agent to use DeepSeek Chat Model as its language model.  
   - Set up prompts or instructions to summarize or process transcription text.  
   - Version 1.9 or later recommended.

5. **Add DeepSeek Chat Model node**  
   - Type: LangChain Language Model (DeepSeek)  
   - Connect as ai_languageModel input to AI Agent node.  
   - Configure DeepSeek API credentials and endpoint.  
   - Ensure compatibility with AI Agent.

6. **Add Google Drive node for transcription**  
   - Type: Google Drive  
   - Connect input from OpenAI node.  
   - Configure Google OAuth2 credentials.  
   - Set to create a Google Docs file with the transcription content.  
   - Set destination folder if desired.

7. **Add Google Drive node for summary**  
   - Type: Google Drive  
   - Connect input from AI Agent node.  
   - Configure Google OAuth2 credentials (can reuse previous).  
   - Set to create a Google Docs file with the summary content.  
   - Set destination folder if desired.

8. **Connect nodes accordingly**  
   - Telegram Trigger → Telegram → OpenAI → AI Agent → Google Drive2  
   - OpenAI → Google Drive

9. **Test the workflow**  
   - Send a voice note to the configured Telegram bot.  
   - Confirm transcription and summary files appear in Google Drive.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                        |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| This workflow depends on valid Telegram bot API credentials and proper webhook setup in Telegram.          | Telegram Bot API Documentation                         |
| OpenAI Whisper model is used for transcription; ensure API quota and billing are configured properly.      | https://platform.openai.com/docs/models/whisper       |
| DeepSeek Chat Model is a LangChain-compatible chat model requiring API credentials and stable network.     | DeepSeek official API docs (vendor-specific)          |
| Google Drive nodes require OAuth2 credentials with permissions to create and edit Google Docs files.       | https://developers.google.com/drive/api/v3/about-auth |
| Ensure all API credentials are stored securely in n8n credentials manager.                                 | n8n official docs on credentials management            |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.