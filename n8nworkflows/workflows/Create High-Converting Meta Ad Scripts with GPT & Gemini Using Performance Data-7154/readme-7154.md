Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data

https://n8nworkflows.xyz/workflows/create-high-converting-meta-ad-scripts-with-gpt---gemini-using-performance-data-7154


# Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data

### 1. Workflow Overview

This workflow automates the process of transcribing podcast audio received via Telegram, generating a structured script outline using OpenAI's GPT models, and saving the results into Notion for further use or reference. It is designed for content creators or marketers who want to efficiently convert audio content into well-organized textual scripts, leveraging AI to enhance productivity.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives voice or audio messages from Telegram users.
- **1.2 Audio Transcription:** Uses an AI model to transcribe the received audio into text.
- **1.3 AI Script Generation:** Processes the transcription to create a script outline with GPT.
- **1.4 Data Persistence:** Saves the generated script outline into Notion for archival or further editing.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block captures incoming Telegram messages via a webhook and routes them for transcription. It serves as the entry point of the workflow.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Telegram

- **Node Details:**  
  1. **Telegram Trigger**  
     - *Type & Role:* Webhook trigger node specialized for Telegram; listens for incoming Telegram updates.  
     - *Configuration:* Uses a unique webhook ID to receive messages from a configured Telegram bot.  
     - *Expressions/Variables:* Automatically captures Telegram message data, including audio files.  
     - *Input/Output:* No input; outputs message data to the Telegram node.  
     - *Edge Cases:* Possible failures include webhook misconfiguration, Telegram API downtime, or invalid bot tokens.  
     - *Sub-workflow:* None.

  2. **Telegram**  
     - *Type & Role:* Telegram node for handling the incoming data; typically downloads audio or confirms message receipt.  
     - *Configuration:* Uses the Telegram bot credentials to interact with the Telegram API.  
     - *Expressions/Variables:* Processes the audio content or relevant message fields.  
     - *Input/Output:* Input from Telegram Trigger; output to Transcribe Audio node.  
     - *Edge Cases:* Possible issues include file download failure, unsupported audio formats, or rate limiting by Telegram.  
     - *Sub-workflow:* None.

#### 2.2 Audio Transcription

- **Overview:**  
  Converts the received audio message into text using OpenAI's language models via the LangChain integration.

- **Nodes Involved:**  
  - Transcribe Audio  
  - OpenAI

- **Node Details:**  
  1. **Transcribe Audio**  
     - *Type & Role:* OpenAI node configured for transcription tasks, possibly using Whisper or GPT audio capabilities.  
     - *Configuration:* Set to transcribe audio content passed from Telegram node.  
     - *Expressions/Variables:* Uses audio file data from Telegram; may specify language or model parameters.  
     - *Input/Output:* Input from Telegram node; output raw transcription text to OpenAI node.  
     - *Edge Cases:* Audio format unsupported, transcription errors, API timeouts, or incorrect audio data passed.  
     - *Sub-workflow:* None.

  2. **OpenAI**  
     - *Type & Role:* OpenAI LangChain node used for further processing or cleanup of transcribed text.  
     - *Configuration:* Likely uses GPT models to refine or validate transcription output.  
     - *Expressions/Variables:* Receives transcription text; may use prompts or instructions to enhance output.  
     - *Input/Output:* Input from Transcribe Audio; output to Code node.  
     - *Edge Cases:* API limits, invalid prompt formatting, or response errors.  
     - *Sub-workflow:* None.

#### 2.3 AI Script Generation

- **Overview:**  
  Processes the cleaned transcription text to generate a high-level script outline for podcast episodes or ads.

- **Nodes Involved:**  
  - Code  
  - Generate Script Outline

- **Node Details:**  
  1. **Code**  
     - *Type & Role:* Function node to manipulate or prepare data for GPT prompt, e.g., formatting or extracting key points.  
     - *Configuration:* Contains JavaScript code to transform input text before sending to GPT.  
     - *Expressions/Variables:* Operates on OpenAI node output, creating structured input for the next node.  
     - *Input/Output:* Input from OpenAI; output to Generate Script Outline.  
     - *Edge Cases:* Code errors, undefined variables, or unexpected input structure.  
     - *Sub-workflow:* None.

  2. **Generate Script Outline**  
     - *Type & Role:* OpenAI LangChain node configured to produce an organized script outline based on input text.  
     - *Configuration:* Uses a prompt template designed to create script structures suitable for Meta ads or podcast summaries.  
     - *Expressions/Variables:* Takes Code node output as prompt input; outputs structured script text.  
     - *Input/Output:* Input from Code; output to Save to Notion node.  
     - *Edge Cases:* Poor prompt design, API failures, or output format inconsistencies.  
     - *Sub-workflow:* None.

#### 2.4 Data Persistence

- **Overview:**  
  Saves the generated script outline into Notion, enabling easy access, editing, and collaboration.

- **Nodes Involved:**  
  - Save to Notion  
  - Save to Notion1

- **Node Details:**  
  1. **Save to Notion**  
     - *Type & Role:* Notion node that creates or updates a page or database item with the generated script outline.  
     - *Configuration:* Connects to a Notion workspace using credentials set up for API access.  
     - *Expressions/Variables:* Uses the script outline text as page content or database property.  
     - *Input/Output:* Input from Generate Script Outline; output to Save to Notion1 node.  
     - *Edge Cases:* Notion API rate limits, credential expiration, insufficient permissions, or data schema mismatches.  
     - *Sub-workflow:* None.

  2. **Save to Notion1**  
     - *Type & Role:* Another Notion node, possibly used for additional saving, confirmation, or writing to a different Notion location.  
     - *Configuration:* Similar to Save to Notion, with potentially different database/page parameters.  
     - *Expressions/Variables:* Acts on previous Notion node output or confirmation.  
     - *Input/Output:* Input from Save to Notion; no further output (end of workflow).  
     - *Edge Cases:* Same as previous Notion node.  
     - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name            | Node Type                     | Functional Role               | Input Node(s)        | Output Node(s)           | Sticky Note                                      |
|----------------------|-------------------------------|------------------------------|----------------------|--------------------------|-------------------------------------------------|
| Telegram Trigger      | n8n-nodes-base.telegramTrigger| Receive Telegram messages     | -                    | Telegram                 |                                                 |
| Telegram             | n8n-nodes-base.telegram        | Handle Telegram message/audio | Telegram Trigger     | Transcribe Audio         |                                                 |
| Transcribe Audio      | @n8n/n8n-nodes-langchain.openAi | Transcribe audio to text      | Telegram              | OpenAI                   |                                                 |
| OpenAI                | @n8n/n8n-nodes-langchain.openAi | Refine transcription output  | Transcribe Audio       | Code                     |                                                 |
| Code                  | n8n-nodes-base.code            | Prepare text for GPT prompt   | OpenAI                | Generate Script Outline   |                                                 |
| Generate Script Outline| @n8n/n8n-nodes-langchain.openAi | Generate structured script    | Code                  | Save to Notion           |                                                 |
| Save to Notion        | n8n-nodes-base.notion          | Save script outline to Notion| Generate Script Outline| Save to Notion1          |                                                 |
| Save to Notion1       | n8n-nodes-base.notion          | Final save/confirmation       | Save to Notion        | -                        |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: `telegramTrigger`  
   - Configure with your Telegram bot API credentials.  
   - Ensure webhook is properly set up to receive messages.  
   - No input connections; output connects to the Telegram node.

2. **Create Telegram node:**  
   - Type: `telegram`  
   - Use the same Telegram credentials.  
   - Configure to handle incoming audio messages (download or access file).  
   - Connect input from Telegram Trigger; output to Transcribe Audio.

3. **Create Transcribe Audio node:**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Configure for audio transcription (e.g., Whisper model or OpenAI audio endpoint).  
   - Pass audio file data from Telegram node input.  
   - Connect output to OpenAI node.

4. **Create OpenAI node:**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Configure with OpenAI API credentials (e.g., API key).  
   - Use for refining or cleaning transcription text.  
   - Input from Transcribe Audio; output to Code node.

5. **Create Code node:**  
   - Type: `code`  
   - Add JavaScript to process the transcription text into a prompt format suitable for script outline generation.  
   - Input from OpenAI node; output to Generate Script Outline node.

6. **Create Generate Script Outline node:**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Configure with OpenAI credentials.  
   - Set prompt template to generate a high-converting Meta ad script outline from the prepared text.  
   - Input from Code node; output to Save to Notion node.

7. **Create Save to Notion node:**  
   - Type: `notion`  
   - Configure Notion API credentials (OAuth2 or integration token).  
   - Set database or page parameters to create a new entry with the script outline.  
   - Input from Generate Script Outline; output to Save to Notion1.

8. **Create Save to Notion1 node:**  
   - Type: `notion`  
   - Configure similarly to Save to Notion, possibly targeting a different database or page for backup or confirmation.  
   - Input from Save to Notion; no output connections.

9. **Connect nodes according to flow:**  
   - Telegram Trigger -> Telegram -> Transcribe Audio -> OpenAI -> Code -> Generate Script Outline -> Save to Notion -> Save to Notion1.

10. **Verify all credentials and API access:**  
   - Telegram Bot credentials with webhook enabled.  
   - OpenAI API key with transcription and GPT model access.  
   - Notion integration with correct database permissions.

11. **Set any default values or constraints:**  
   - Audio file format constraints (e.g., mp3, wav) in Telegram node.  
   - Timeout and retry settings for OpenAI nodes.  
   - Proper error handling in Code node for unexpected input.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                         |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| This workflow leverages OpenAI’s LangChain integration for transcription and text generation.   | See https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.langchain/                            |
| Telegram bot setup with webhook is required to receive audio messages from users.                | Telegram Bot API documentation: https://core.telegram.org/bots/api                                        |
| Notion API integration requires creating an integration and sharing databases/pages with it.    | Notion API documentation: https://developers.notion.com/docs/getting-started                            |
| The workflow is designed for podcast creators looking to automate transcription and script prep.| Useful for creating high-converting Meta ad scripts from spoken content.                               |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.