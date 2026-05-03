Convert Telegram Voice Messages to Professional Emails with Whisper, GPT-4 & Gmail

https://n8nworkflows.xyz/workflows/convert-telegram-voice-messages-to-professional-emails-with-whisper--gpt-4---gmail-7041


# Convert Telegram Voice Messages to Professional Emails with Whisper, GPT-4 & Gmail

### 1. Workflow Overview

This workflow automates the conversion of Telegram voice messages into professionally composed emails sent via Gmail. It targets users who receive voice notes on Telegram and want an efficient, AI-powered pipeline to transcribe, interpret, and compose emails based on the voice content. The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Triggered by incoming Telegram voice messages.
- **1.2 Pre-Processing Buffer:** Introduces wait/delay nodes to handle message bursts and processing stability.
- **1.3 Audio Retrieval and Transcription:** Downloads the voice file and uses OpenAI Whisper to transcribe audio to text.
- **1.4 AI Email Content Generation:** Utilizes a Langchain AI Agent combined with GPT-4 to analyze the transcription, detect recipient email addresses, understand intent, and generate a structured HTML email.
- **1.5 Email Validation and Sending:** Enforces strict JSON schema validation of AI output and sends the email via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for Telegram voice messages sent to the bot and triggers the workflow when a message arrives.

**Nodes Involved:**  
- Telegram Voice Message Trigger

**Node Details:**  

| Node Name                      | Details                                                                                                         |
|-------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Telegram Voice Message Trigger | - Type: Telegram Trigger node, listens for new messages.  
- Configured to trigger on `message` updates only.  
- Credential: Linked to a Telegram Bot API token credential.  
- Input: Incoming Telegram messages.  
- Output: Passes the full message JSON downstream.  
- Edge cases: Only one Telegram trigger node per bot allowed; failure if bot credential invalid or Telegram API down. |

---

#### 2.2 Pre-Processing Buffer

**Overview:**  
Introduces a brief delay after trigger reception to prevent race conditions or overload from simultaneous incoming messages.

**Nodes Involved:**  
- Buffer Delay (Pre-Processing)

**Node Details:**  

| Node Name               | Details                                                                                                         |
|------------------------|-----------------------------------------------------------------------------------------------------------------|
| Buffer Delay (Pre-Processing) | - Type: Wait node, no parameters set (default wait duration).  
- Input: From Telegram trigger.  
- Output: Passes data downstream after delay.  
- Edge cases: Minimal risk; ensures workflow stability on bursts. |

---

#### 2.3 Audio Retrieval and Transcription

**Overview:**  
Downloads the Telegram voice message file and transcribes it into text using OpenAI Whisper.

**Nodes Involved:**  
- Download Telegram Voice File  
- Transcribe Audio to Text (OpenAI Whisper)  
- Buffer Delay (Pre-AI Processing)

**Node Details:**  

| Node Name                   | Details                                                                                                         |
|-----------------------------|-----------------------------------------------------------------------------------------------------------------|
| Download Telegram Voice File | - Type: Telegram node with `file` resource.  
- Downloads voice message file using `file_id` extracted from incoming message JSON.  
- Input: Receives Telegram message JSON from buffer delay.  
- Output: Passes downloaded audio file to transcription node.  
- Edge cases: Failures if file ID missing or Telegram API fails. |
| Transcribe Audio to Text (OpenAI Whisper) | - Type: Langchain OpenAI node configured for `audio` resource and `transcribe` operation.  
- Credential: OpenAI API key.  
- Sends audio file to Whisper for transcription.  
- Output: Returns transcribed text.  
- Edge cases: API quota exceeded, timeouts, poor audio quality affecting transcription accuracy. |
| Buffer Delay (Pre-AI Processing) | - Type: Wait node, no parameters (default delay).  
- Ensures transcription completes before AI processing.  
- Input: From transcription node.  
- Output: Passes transcribed text downstream. |

---

#### 2.4 AI Email Content Generation

**Overview:**  
Processes the transcribed text with an AI Agent to detect recipient email, understand intent, generate a professional email subject and HTML body, using GPT-4 as the language model.

**Nodes Involved:**  
- Generate Email Content AI AGENT  
- GPT-4 Email Generator Model  
- Enforce Email JSON Schema

**Node Details:**  

| Node Name                      | Details                                                                                                         |
|-------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Generate Email Content AI AGENT | - Type: Langchain AI Agent node.  
- Uses a custom prompt instructing the agent to:  
  - Detect email recipient (completes partial addresses with `@gmail.com`).  
  - Understand user intent (e.g., resignation, request).  
  - Generate concise email subject.  
  - Write professional HTML email body with polite tone and no spelling mistakes.  
- Input expression uses `{{ $json.text }}` representing transcribed text.  
- Output parsed by a structured output parser downstream.  
- Connects to GPT-4 Model node for language generation.  
- Edge cases: Improper transcription input, prompt misinterpretation, OpenAI API failures. |
| GPT-4 Email Generator Model    | - Type: Langchain OpenAI Chat Model node.  
- Model: `gpt-4.1-mini` variant selected.  
- Credential: OpenAI API key.  
- Serves as backend language model for AI Agent node.  
- Edge cases: API rate limits, model downtime, network errors. |
| Enforce Email JSON Schema      | - Type: Langchain structured output parser.  
- Validates AI output strictly follows JSON schema with keys: `email`, `subject`, `body`.  
- Ensures output is safe and properly formatted for email sending.  
- Edge cases: AI output not matching schema, JSON parsing errors. |

---

#### 2.5 Email Validation and Sending

**Overview:**  
Sends the generated email to the detected recipient via Gmail.

**Nodes Involved:**  
- Send Email

**Node Details:**  

| Node Name  | Details                                                                                                         |
|------------|-----------------------------------------------------------------------------------------------------------------|
| Send Email | - Type: Gmail node.  
- Parameters:  
  - To: `{{$json.output.email}}` (email from AI JSON output).  
  - Subject: `{{$json.output.subject}}`.  
  - Message Body (HTML): `{{$json.output.body}}`.  
- Credential: Gmail OAuth2 credential configured for sending emails.  
- Edge cases: Gmail API authentication errors, quota limits, invalid email addresses, network issues. |

---

### 3. Summary Table

| Node Name                        | Node Type                               | Functional Role                          | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                          |
|---------------------------------|---------------------------------------|----------------------------------------|---------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------|
| Telegram Voice Message Trigger   | Telegram Trigger                      | Input reception from Telegram voice messages | —                               | Buffer Delay (Pre-Processing)      | See note: Node Setup – Initial Steps (Telegram → Download Audio)                                   |
| Buffer Delay (Pre-Processing)    | Wait                                 | Buffer incoming messages to avoid overload | Telegram Voice Message Trigger  | Download Telegram Voice File       | See note: Node Setup – Initial Steps (Telegram → Download Audio)                                   |
| Download Telegram Voice File     | Telegram                             | Download voice audio file from Telegram | Buffer Delay (Pre-Processing)   | Transcribe Audio to Text (OpenAI Whisper) | See note: Node Setup – Initial Steps (Telegram → Download Audio)                                   |
| Transcribe Audio to Text (OpenAI Whisper) | Langchain OpenAI (audio transcribe) | Transcribe audio to text using Whisper | Download Telegram Voice File    | Buffer Delay (Pre-AI Processing)  | See note: Node Setup – AI Transcription & Email Generation                                         |
| Buffer Delay (Pre-AI Processing) | Wait                                | Buffer before AI processing             | Transcribe Audio to Text (OpenAI Whisper) | Generate Email Content AI AGENT  | See note: Node Setup – AI Transcription & Email Generation                                         |
| Generate Email Content AI AGENT  | Langchain AI Agent                   | Generate professional email content from transcription | Buffer Delay (Pre-AI Processing) | Send Email                       | See note: Node Setup – AI Transcription & Email Generation                                         |
| GPT-4 Email Generator Model      | Langchain OpenAI Chat Model          | Backend GPT-4 language model for AI Agent | —                               | Generate Email Content AI AGENT   | See note: Node Setup – AI Transcription & Email Generation                                         |
| Enforce Email JSON Schema        | Langchain Output Parser Structured   | Validate AI JSON output schema          | Generate Email Content AI AGENT | Send Email                       | See note: Node Setup – AI Transcription & Email Generation                                         |
| Send Email                      | Gmail                               | Send the generated email via Gmail     | Generate Email Content AI AGENT | —                                 | See note: Node Setup – AI Transcription & Email Generation                                         |
| Sticky Note                     | Sticky Note                         | Documentation notes                     | —                               | —                                 | Multiple nodes covered by content in initial sticky notes                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Voice Message Trigger node**  
   - Node type: Telegram Trigger  
   - Set credential to your Telegram Bot API credential.  
   - Trigger on `message` updates only.  
   - Position this as the workflow entry node.

2. **Create Buffer Delay (Pre-Processing) node**  
   - Node type: Wait  
   - No parameters needed (default delay).  
   - Connect Telegram Trigger output to this node.

3. **Create Download Telegram Voice File node**  
   - Node type: Telegram  
   - Resource: `file`  
   - Parameter `fileId`: set to `{{$json["message"]["voice"]["file_id"]}}` (extract file_id from incoming message).  
   - Connect Buffer Delay (Pre-Processing) output to this node.

4. **Create Transcribe Audio to Text (OpenAI Whisper) node**  
   - Node type: Langchain OpenAI  
   - Resource: `audio`  
   - Operation: `transcribe`  
   - Assign OpenAI API credential (your OpenAI key).  
   - Connect Download Telegram Voice File output to this node.

5. **Create Buffer Delay (Pre-AI Processing) node**  
   - Node type: Wait  
   - No parameters (default delay).  
   - Connect Transcribe Audio to Text output to this node.

6. **Create GPT-4 Email Generator Model node**  
   - Node type: Langchain OpenAI Chat Model  
   - Model: Select `gpt-4.1-mini` or equivalent GPT-4 variant.  
   - Assign OpenAI API credential.  
   - This node will be linked as the languageModel for the AI Agent node.

7. **Create Enforce Email JSON Schema node**  
   - Node type: Langchain Output Parser Structured  
   - Paste the JSON schema example:  
     ```json
     {
       "email": "exemple@gmail.com",
       "subject": "Objet de l’email",
       "body": "<p>Bonjour,</p><p>Voici un exemple d’email professionnel bien structuré avec des balises HTML.</p><p>Cordialement,<br>Jean Dupont</p>"
     }
     ```  
   - Connect as the output parser for the AI Agent node.

8. **Create Generate Email Content AI AGENT node**  
   - Node type: Langchain AI Agent  
   - Prompt type: `define`  
   - In the prompt text, instruct the agent to:  
     - Detect recipient email (complete partial addresses with `@gmail.com`).  
     - Understand intent (e.g., resignation, request).  
     - Generate concise email subject.  
     - Write professional HTML-formatted email body, polite tone, no spelling mistakes.  
     - Use variable `{{ $json.text }}` to pass the transcription text.  
     - Add custom note: “my first name is Jeremy and if recipient is unhappy I resign”.  
   - Link the AI Agent node to:  
     - GPT-4 Email Generator Model node as languageModel.  
     - Enforce Email JSON Schema node as outputParser.  
   - Connect Buffer Delay (Pre-AI Processing) output to this node.

9. **Create Send Email node**  
   - Node type: Gmail  
   - Set parameters:  
     - To: `{{$json.output.email}}`  
     - Subject: `{{$json.output.subject}}`  
     - Message: `{{$json.output.body}}` (HTML content)  
   - Assign Gmail OAuth2 credential for sending.  
   - Connect Generate Email Content AI AGENT output to this node.

10. **Verify all connections**  
    - Telegram Voice Message Trigger → Buffer Delay (Pre-Processing) → Download Telegram Voice File → Transcribe Audio to Text → Buffer Delay (Pre-AI Processing) → Generate Email Content AI AGENT → Send Email  
    - GPT-4 Email Generator Model and Enforce Email JSON Schema linked as language model and output parser to AI Agent node respectively.

---

### 5. General Notes & Resources

| Note Content                                                                                                                     | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Only one Telegram Trigger node per Telegram Bot is supported.                                                                    | Telegram API constraint                                                                             |
| Buffer Delay nodes help avoid issues with handling multiple simultaneous Telegram messages or asynchronous API responses.         | Stability and reliability of workflow                                                              |
| OpenAI Whisper transcription quality depends on audio clarity and API limits.                                                    | https://openai.com/blog/whisper                                                                    |
| GPT-4 model variant `gpt-4.1-mini` used for cost-effective yet powerful language generation.                                     | https://openai.com/api/models                                                                      |
| Gmail node requires OAuth2 credentials properly configured with sending rights.                                                  | https://developers.google.com/gmail/api/auth/about-auth                                            |
| The AI Agent prompt is carefully designed to produce strict JSON output to facilitate parsing and safe email sending.           | See node “Generate Email Content AI AGENT” prompt details above                                    |
| Workflow designed for French language content, including polite and formal tone in emails.                                       | Adapt prompt accordingly for other languages or tones                                             |
| Sticky Notes in the workflow provide detailed setup instructions and explanations for initial nodes and AI processing section.  | Available inside the workflow editor for user reference                                            |

---

**Disclaimer:**  
The text provided is sourced exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.