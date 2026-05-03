Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data

https://n8nworkflows.xyz/workflows/create-high-converting-meta-ad-scripts-with-gpt---gemini-using-performance-data-7157


# Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data

---

### 1. Workflow Overview

This n8n workflow, titled **"Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data"** (though the actual JSON corresponds to a "Podcast Note taker"), is designed to automate the transcription and summarization of podcast audio messages received via Telegram, and then save the generated notes to Notion. It integrates AI-powered transcription and summarization using OpenAI (Langchain nodes) and organizes the data for easy access in a Notion database.

**Target Use Cases:**  
- Podcast creators who want automated transcription and note-taking from audio clips sent via Telegram.  
- Content teams aiming to streamline podcast content processing and archiving in Notion.  
- Users leveraging AI transcription and summarization to generate concise script outlines or show notes.

**Logical Blocks:**  
- **1.1 Input Reception via Telegram**: Receiving audio input through Telegram trigger and bot nodes.  
- **1.2 AI Transcription & Processing**: Transcribing audio and generating a script outline via OpenAI-powered nodes, including intermediate code logic.  
- **1.3 Data Storage in Notion**: Saving the processed script outline and transcription data into Notion databases for record-keeping and further use.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception via Telegram

- **Overview:**  
  This block handles receiving audio input from users via Telegram, triggering the workflow when a message is received, and forwarding the message for processing.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Telegram

- **Node Details:**  

  **Telegram Trigger**  
  - Type: Telegram Trigger node  
  - Role: Entry point that listens for incoming Telegram messages (audio or voice notes).  
  - Configuration: Uses a webhook ID to capture Telegram updates. No additional filters specified, so it likely triggers on all messages.  
  - Inputs: None (trigger node).  
  - Outputs: Sends the message object to the Telegram node.  
  - Edge Cases: Potential webhook connectivity issues, message type mismatches, or Telegram API rate limits.

  **Telegram**  
  - Type: Telegram node  
  - Role: Acts as a Telegram Bot to interact with incoming messages (e.g., acknowledge receipt or request).  
  - Configuration: Uses a webhook ID and credentials linked to a Telegram bot.  
  - Inputs: Receives data from Telegram Trigger.  
  - Outputs: Sends the audio data or message details to the next node (Transcribe Audio).  
  - Edge Cases: Authentication errors with Telegram API, message format issues, or network timeouts.

---

#### 1.2 AI Transcription & Processing

- **Overview:**  
  This block converts the audio message into text, processes it through OpenAI models to generate a transcript and a summarized script outline, potentially applying custom code logic between these steps.

- **Nodes Involved:**  
  - Transcribe Audio  
  - OpenAI  
  - Code  
  - Generate Script Outline

- **Node Details:**

  **Transcribe Audio**  
  - Type: OpenAI Langchain node (likely using Whisper or similar ASR model)  
  - Role: Transcribe the audio content received from Telegram into text.  
  - Configuration: Connects to OpenAI API with transcription parameters set for audio input.  
  - Inputs: Audio data from Telegram node.  
  - Outputs: Transcribed text sent to OpenAI node for further processing.  
  - Edge Cases: Audio format unsupported, transcription errors, API quota limits, or timeouts.

  **OpenAI**  
  - Type: OpenAI Langchain node  
  - Role: Processes raw transcribed text, possibly to clean or enrich before feeding into code logic.  
  - Configuration: Calls OpenAI API with prompt or model parameters to parse or enhance the transcript.  
  - Inputs: Transcribed text from Transcribe Audio.  
  - Outputs: Sends processed text to Code node.  
  - Edge Cases: API errors, prompt failures, rate limits.

  **Code**  
  - Type: Code node (JavaScript)  
  - Role: Implements custom logic or data transformation on the processed text (e.g., formatting, filtering).  
  - Configuration: Custom script embedded in node parameters.  
  - Inputs: OpenAI processed text.  
  - Outputs: Passes manipulated data to Generate Script Outline node.  
  - Edge Cases: Script runtime errors, undefined variables, expression errors.

  **Generate Script Outline**  
  - Type: OpenAI Langchain node  
  - Role: Takes finalized transcript input and generates a concise script outline or summary for the podcast episode.  
  - Configuration: Calls OpenAI model with a prompt tailored to produce high-converting ad scripts or podcast notes.  
  - Inputs: Data from Code node.  
  - Outputs: Sends generated outlines to Notion storage.  
  - Edge Cases: Model response errors, incomplete outputs, API limits.

---

#### 1.3 Data Storage in Notion

- **Overview:**  
  This block saves the generated script outlines and related notes into Notion databases for record keeping and future reference.

- **Nodes Involved:**  
  - Save to Notion  
  - Save to Notion1

- **Node Details:**

  **Save to Notion**  
  - Type: Notion node  
  - Role: Inserts or updates a Notion database/page with the generated script outline or transcription data.  
  - Configuration: Requires Notion credentials and target database/page ID.  
  - Inputs: Receives script outline from Generate Script Outline.  
  - Outputs: Passes data to Save to Notion1 node for possible further saving or confirmation.  
  - Edge Cases: Authentication failure, permission denied, invalid database ID, network errors.

  **Save to Notion1**  
  - Type: Notion node  
  - Role: Secondary save or update step for Notion, potentially refining or adding additional metadata.  
  - Configuration: Similar to Save to Notion node, configured with Notion credentials and database/page info.  
  - Inputs: From Save to Notion node.  
  - Outputs: None (end of workflow).  
  - Edge Cases: Same as Save to Notion node.

---

### 3. Summary Table

| Node Name             | Node Type                      | Functional Role                     | Input Node(s)       | Output Node(s)          | Sticky Note                |
|-----------------------|--------------------------------|-----------------------------------|---------------------|-------------------------|----------------------------|
| Telegram Trigger       | Telegram Trigger               | Receives Telegram messages (entry)| None                | Telegram                |                            |
| Telegram              | Telegram                      | Handles Telegram bot interactions | Telegram Trigger    | Transcribe Audio        |                            |
| Transcribe Audio       | OpenAI Langchain (Transcription) | Converts audio to text             | Telegram             | OpenAI                  |                            |
| OpenAI                | OpenAI Langchain              | Processes transcribed text         | Transcribe Audio     | Code                    |                            |
| Code                  | Code (JavaScript)             | Custom data transformation         | OpenAI               | Generate Script Outline |                            |
| Generate Script Outline| OpenAI Langchain              | Creates summarized script outline  | Code                 | Save to Notion          |                            |
| Save to Notion         | Notion                       | Saves outline to Notion database   | Generate Script Outline | Save to Notion1        |                            |
| Save to Notion1        | Notion                       | Further saves or updates Notion    | Save to Notion       | None                    |                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure with Telegram Bot credentials and set webhook to receive all message types (or specifically audio/voice).  
   - This node listens for incoming Telegram messages and triggers the workflow.

2. **Create Telegram node**  
   - Type: Telegram  
   - Connect input from Telegram Trigger node.  
   - Configure with same Telegram Bot credentials to manage message handling.  
   - This node forwards audio data for transcription.

3. **Create Transcribe Audio node**  
   - Type: OpenAI Langchain (or equivalent transcription node)  
   - Connect input from Telegram node.  
   - Configure OpenAI credentials (API key).  
   - Set to transcribe audio input formats (e.g., `.ogg`, `.mp3`).  
   - Configure any language options or model parameters as needed.

4. **Create OpenAI node**  
   - Type: OpenAI Langchain  
   - Connect input from Transcribe Audio node.  
   - Configure OpenAI API with prompt to process or clean the raw transcript.  
   - Set model version and temperature as required for text processing.

5. **Create Code node**  
   - Type: Code (JavaScript)  
   - Connect input from OpenAI node.  
   - Enter custom JavaScript code to manipulate or reformat the processed transcript.  
   - Example: parsing JSON, trimming text, or adding metadata.

6. **Create Generate Script Outline node**  
   - Type: OpenAI Langchain  
   - Connect input from Code node.  
   - Configure prompt to generate a concise script outline or summary suitable for ad scripts or podcast notes.  
   - Set model parameters (e.g., GPT-4 or GPT-3.5, prompt engineering).  

7. **Create Save to Notion node**  
   - Type: Notion  
   - Connect input from Generate Script Outline node.  
   - Configure Notion credentials (OAuth2 or Integration Token).  
   - Set target database or page ID to store the script outline data.  
   - Map fields such as title, content, and metadata accordingly.

8. **Create Save to Notion1 node**  
   - Type: Notion  
   - Connect input from Save to Notion node.  
   - Configure similarly with Notion credentials and target database/page.  
   - Use to save additional data or update existing entries if required.  

9. **Set execution order and test**  
   - Validate connections and credentials.  
   - Test workflow by sending an audio message to the configured Telegram bot.  
   - Confirm transcription, outline generation, and data storage in Notion.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                 |
|-----------------------------------------------------------------------------------------------|------------------------------------------------|
| Workflow tags include: ai, podcast, notion — indicating the focus on AI-powered podcast note-taking and Notion integration | Tags from workflow metadata                      |
| The workflow uses OpenAI Langchain nodes for transcription and text generation — requiring valid OpenAI API credentials and quota | OpenAI API documentation: https://platform.openai.com/docs/ |
| Telegram nodes require bot credentials and proper webhook setup for real-time message capture | Telegram Bot API docs: https://core.telegram.org/bots/api |
| Notion nodes need an integration token or OAuth2 credentials with access to the target database | Notion API docs: https://developers.notion.com/ |
| Potential failure points include API rate limits, network issues, and data format mismatches | General integration risk notes                   |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing respects all applicable content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.