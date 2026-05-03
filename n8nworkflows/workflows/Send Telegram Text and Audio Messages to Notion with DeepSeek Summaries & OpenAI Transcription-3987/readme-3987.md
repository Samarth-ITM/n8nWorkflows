Send Telegram Text and Audio Messages to Notion with DeepSeek Summaries & OpenAI Transcription

https://n8nworkflows.xyz/workflows/send-telegram-text-and-audio-messages-to-notion-with-deepseek-summaries---openai-transcription-3987


# Send Telegram Text and Audio Messages to Notion with DeepSeek Summaries & OpenAI Transcription

### 1. Workflow Overview

This workflow automates the capture, processing, and organization of Telegram messages (text, voice notes, audio files) into Notion databases using AI-powered summarization and transcription services. It targets users who want to streamline communication and task management by integrating Telegram directly with Notion, leveraging DeepSeek for text summarization and OpenAI for audio transcription.

**Logical Blocks:**

- **1.1 Input Reception:** Captures incoming Telegram messages via webhook.
- **1.2 Message Type Routing:** Differentiates message types (text, voice note, audio file) to route processing accordingly.
- **1.3 Text Message Processing:** Uses DeepSeek to summarize text messages and stores summaries in Notion "Tasks Tracker."
- **1.4 Audio Transcription Processing:** Uses OpenAI transcription on voice notes and audio files; saves transcriptions to Notion "Transcribes" database.
- **1.5 AI Agent Orchestration:** Coordinates AI models (DeepSeek and OpenAI) for specialized processing tasks.
- **1.6 Notion Integration:** Updates appropriate Notion databases with processed content for task and transcription tracking.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives all incoming Telegram messages from a configured chat via a Telegram trigger node set up with webhook.

**Nodes Involved:**  
- Telegram Trigger

**Node Details:**  
- **Telegram Trigger**  
  - *Type:* Telegram Trigger Node  
  - *Role:* Listens for new messages in a Telegram chat via webhook.  
  - *Configuration:* Default parameters; webhook ID configured for Telegram bot integration.  
  - *Inputs:* External webhook from Telegram platform.  
  - *Outputs:* Emits incoming message data to next node for processing.  
  - *Edge Cases:*  
    - Telegram bot connectivity issues or webhook misconfiguration.  
    - Message types outside expected (text, voice, audio) may require filtering downstream.  
  - *Version:* 1

---

#### 2.2 Message Type Routing

**Overview:**  
Determines the type of incoming Telegram message and routes the flow to appropriate processing: text messages, voice notes/audio, or other.

**Nodes Involved:**  
- Switch1

**Node Details:**  
- **Switch1**  
  - *Type:* Switch Node  
  - *Role:* Branches workflow based on message content type.  
  - *Configuration:* Conditions based on Telegram message attributes, likely distinguishing text messages, voice notes, and audio files.  
  - *Inputs:* Incoming message data from Telegram Trigger.  
  - *Outputs:*  
    - Output 1: Routed to Telegram node for audio transcription processing.  
    - Output 2: Routed to AI Agent for text summarization.  
    - Output 3: Routed to Telegram1 node for alternative audio transcription.  
  - *Edge Cases:*  
    - Unrecognized message types may not be handled.  
    - Misclassification due to unexpected message formats.  
  - *Version:* 3.2

---

#### 2.3 Text Message Processing

**Overview:**  
Processes text messages by summarizing them with DeepSeek AI and then posting summarized tasks to Notion.

**Nodes Involved:**  
- AI Agent  
- DeepSeek Chat Model  
- Notion1

**Node Details:**  
- **AI Agent**  
  - *Type:* LangChain AI Agent Node  
  - *Role:* Orchestrates AI model interactions for text summarization tasks.  
  - *Configuration:* Receives messages routed for text summarization; interfaces with DeepSeek Chat Model as language model.  
  - *Inputs:* Output 2 from Switch1.  
  - *Outputs:* Summarized text data to Notion1.  
  - *Edge Cases:* AI model response delays or failures.  
  - *Version:* 1.9

- **DeepSeek Chat Model**  
  - *Type:* LangChain DeepSeek Language Model Node  
  - *Role:* Provides AI summarization of input text messages.  
  - *Configuration:* Default DeepSeek chat model settings (likely fine-tuned for summarization).  
  - *Inputs:* Connected internally from AI Agent (as ai_languageModel).  
  - *Outputs:* Summarized text to AI Agent.  
  - *Edge Cases:* API rate limits or summarization inaccuracies.  
  - *Version:* 1

- **Notion1**  
  - *Type:* Notion Node  
  - *Role:* Inserts summarized text message entries into "Tasks Tracker" Notion database.  
  - *Configuration:* Configured with Notion credentials and target database; maps text message as title and summary as description; adds current date.  
  - *Inputs:* Summarized output from AI Agent.  
  - *Outputs:* None (final storage step).  
  - *Edge Cases:* Authentication failures, database permission issues, or incorrect mapping.  
  - *Version:* 2.2

---

#### 2.4 Audio Transcription Processing

**Overview:**  
Handles voice notes and audio files by transcribing them using OpenAI and saving the transcriptions in Notion "Transcribes" database.

**Nodes Involved:**  
- Telegram  
- OpenAI  
- Notion  
- Telegram1  
- OpenAI2  
- Notion2

**Node Details:**  
- **Telegram**  
  - *Type:* Telegram Node  
  - *Role:* Retrieves voice note or audio file content for transcription.  
  - *Configuration:* Set up with Telegram credentials; processes first audio type route from Switch1.  
  - *Inputs:* Output 1 from Switch1.  
  - *Outputs:* Audio data forwarded to OpenAI transcription node.  
  - *Edge Cases:* Media download failures, Telegram API limits.  
  - *Version:* 1.2

- **OpenAI**  
  - *Type:* LangChain OpenAI Node  
  - *Role:* Transcribes audio file to text using OpenAI's transcription API.  
  - *Configuration:* Uses OpenAI credentials and transcription model (e.g., Whisper).  
  - *Inputs:* Audio content from Telegram node.  
  - *Outputs:* Transcribed text to Notion node.  
  - *Edge Cases:* Transcription errors, API timeouts, or quota exhaustion.  
  - *Version:* 1

- **Notion**  
  - *Type:* Notion Node  
  - *Role:* Saves transcription text to "Transcribes" Notion database.  
  - *Configuration:* Maps transcription text and relevant metadata; configured for target Notion database.  
  - *Inputs:* Transcribed text from OpenAI.  
  - *Outputs:* None (final storage step).  
  - *Edge Cases:* Permission or API errors.  
  - *Version:* 2.1

- **Telegram1**  
  - *Type:* Telegram Node  
  - *Role:* Handles alternative audio input route from Switch1 for transcription.  
  - *Configuration:* Similar to Telegram node for first audio path, but separate webhook.  
  - *Inputs:* Output 3 from Switch1.  
  - *Outputs:* Forwards audio to OpenAI2.  
  - *Edge Cases:* Same as Telegram node.  
  - *Version:* 1.2

- **OpenAI2**  
  - *Type:* LangChain OpenAI Node  
  - *Role:* Transcribes alternative audio inputs.  
  - *Configuration:* Same as OpenAI node but separate instance.  
  - *Inputs:* Audio content from Telegram1.  
  - *Outputs:* Transcription to Notion2.  
  - *Edge Cases:* Same transcription API risks.  
  - *Version:* 1

- **Notion2**  
  - *Type:* Notion Node  
  - *Role:* Stores alternative audio transcription results.  
  - *Configuration:* Similar to Notion node, targets correct Notion database.  
  - *Inputs:* Output from OpenAI2.  
  - *Outputs:* None.  
  - *Edge Cases:* Same as Notion node.  
  - *Version:* 2.1

---

#### 2.5 Sticky Notes

**Overview:**  
Several sticky notes are present but contain empty content; likely placeholders for future comments or instructions.

**Nodes Involved:**  
- Sticky Note (four instances)

**Node Details:**  
- **Sticky Note1, Sticky Note2, Sticky Note3, Sticky Note4**  
  - *Type:* Sticky Note Node  
  - *Role:* Visual annotations or reminders.  
  - *Configuration:* Empty content; no functional impact.  
  - *Inputs/Outputs:* None.

---

### 3. Summary Table

| Node Name           | Node Type                       | Functional Role                    | Input Node(s)    | Output Node(s)     | Sticky Note |
|---------------------|--------------------------------|----------------------------------|------------------|--------------------|-------------|
| Telegram Trigger     | Telegram Trigger                | Receives Telegram messages       | -                | Switch1            |             |
| Switch1             | Switch                         | Routes messages by type           | Telegram Trigger | Telegram, AI Agent, Telegram1 |             |
| AI Agent            | LangChain AI Agent             | Orchestrates text summarization  | Switch1          | Notion1            |             |
| DeepSeek Chat Model  | LangChain DeepSeek LM          | Summarizes text messages         | AI Agent (lmChat) | AI Agent           |             |
| Notion1             | Notion                         | Saves summarized tasks           | AI Agent         | -                  |             |
| Telegram            | Telegram                       | Retrieves audio file (path 1)    | Switch1          | OpenAI             |             |
| OpenAI              | LangChain OpenAI               | Transcribes audio to text (path 1)| Telegram         | Notion             |             |
| Notion              | Notion                         | Saves transcription (path 1)    | OpenAI           | -                  |             |
| Telegram1           | Telegram                       | Retrieves audio file (path 2)    | Switch1          | OpenAI2            |             |
| OpenAI2             | LangChain OpenAI               | Transcribes audio to text (path 2)| Telegram1        | Notion2            |             |
| Notion2             | Notion                         | Saves transcription (path 2)    | OpenAI2          | -                  |             |
| Sticky Note1        | Sticky Note                    | Visual annotation (empty)        | -                | -                  |             |
| Sticky Note2        | Sticky Note                    | Visual annotation (empty)        | -                | -                  |             |
| Sticky Note3        | Sticky Note                    | Visual annotation (empty)        | -                | -                  |             |
| Sticky Note4        | Sticky Note                    | Visual annotation (empty)        | -                | -                  |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Telegram Trigger node:**
   - Type: Telegram Trigger  
   - Configure with your Telegram bot credentials.  
   - Set webhook for receiving messages from your target Telegram chat.

2. **Add a Switch node ("Switch1"):**
   - Type: Switch  
   - Connect input from Telegram Trigger node.  
   - Configure conditions to differentiate message types:  
     - Output 1: For voice notes/audio file messages (e.g., check message.media_type or similar).  
     - Output 2: For text messages.  
     - Output 3: For alternative audio cases or fallback.

3. **Text Message Path:**
   - Add a LangChain AI Agent node ("AI Agent"):  
     - Connect input from Switch1 output 2.  
     - Configure agent to use a DeepSeek chat model.  
   - Add LangChain DeepSeek Chat Model node ("DeepSeek Chat Model"):  
     - Connect as the language model for the AI Agent node.  
     - Use default or fine-tuned DeepSeek model settings for summarization.  
   - Add Notion node ("Notion1"):  
     - Connect input from AI Agent output.  
     - Configure with Notion credentials and select "Tasks Tracker" database.  
     - Map: Original message text as title, DeepSeek summary as description, current date as date property.

4. **Audio Transcription Path 1:**
   - Add Telegram node ("Telegram"):  
     - Connect input from Switch1 output 1.  
     - Configure with Telegram credentials to retrieve audio file content.  
   - Add LangChain OpenAI node ("OpenAI"):  
     - Connect input from Telegram node output.  
     - Configure with OpenAI credentials using Whisper or equivalent transcription model.  
   - Add Notion node ("Notion"):  
     - Connect input from OpenAI output.  
     - Configure targeting "Transcribes" Notion database.  
     - Map transcription text and relevant metadata.

5. **Audio Transcription Path 2 (alternative):**
   - Add Telegram node ("Telegram1"):  
     - Connect input from Switch1 output 3.  
     - Configure similarly to Telegram node for audio input.  
   - Add LangChain OpenAI node ("OpenAI2"):  
     - Connect input from Telegram1 output.  
     - Configure with OpenAI credentials for transcription.  
   - Add Notion node ("Notion2"):  
     - Connect input from OpenAI2 output.  
     - Configure for "Transcribes" Notion database.  
     - Map transcription data accordingly.

6. **Add optional Sticky Notes for documentation or reminders (content optional).**

7. **Verify Connections:**
   - Ensure Telegram Trigger outputs to Switch1.  
   - Switch1 routes properly to all processing paths (text and audio).  
   - AI Agent uses DeepSeek Chat Model as language model.  
   - OpenAI nodes configured for transcription.  
   - Notion nodes targeting correct databases with proper property mappings.

8. **Credential Setup:**
   - Telegram: OAuth2 or Bot Token.  
   - OpenAI: API Key with transcription access.  
   - Notion: Integration Token with write access to specified databases.  
   - LangChain AI nodes: linked with OpenAI and DeepSeek credentials.

9. **Test the workflow:**
   - Send text message in Telegram → check summarized task in Notion.  
   - Send voice note/audio → check transcription created in Notion.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                  |
|-----------------------------------------------------------------------------------------------|-------------------------------------------------|
| This workflow uses DeepSeek for advanced text summarization, enhancing clarity and brevity.   | DeepSeek AI usage                                |
| OpenAI Whisper model is leveraged for accurate audio transcription of voice notes and files. | OpenAI Whisper API documentation                 |
| Integration supports multiple audio paths to handle diverse Telegram audio message formats.   | Telegram Bot API media handling                   |
| Notion databases "Tasks Tracker" and "Transcribes" must exist with proper schema for mapping.| Notion database setup instructions                |

---

**Disclaimer:** The provided description and workflow are generated entirely from an automated n8n workflow export. The content adheres to all applicable content policies and contains no illegal, offensive, or protected material. All handled data is legal and publicly accessible.