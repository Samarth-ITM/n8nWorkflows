Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data

https://n8nworkflows.xyz/workflows/create-high-converting-meta-ad-scripts-with-gpt---gemini-using-performance-data-7142


# Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data

### 1. Workflow Overview

This workflow is designed to automate the process of creating structured podcast notes from audio content received via Telegram, leveraging AI transcription and text generation capabilities, and storing the results in Notion for easy access and organization. The workflow targets users who want to streamline podcast content management and note-taking by integrating Telegram, OpenAI/GPT-based transcription and script generation, and Notion as a knowledge base.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives audio input via Telegram messages.
- **1.2 AI Transcription:** Converts the audio message into text using OpenAI transcription.
- **1.3 AI Script Generation:** Processes the transcribed text to generate a structured script outline via OpenAI.
- **1.4 Notion Storage:** Saves the generated script outline and the raw transcript into Notion for record-keeping and future reference.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block handles reception of incoming audio data from users via Telegram. It uses a trigger node to capture Telegram messages and passes them to the next node for transcription.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Telegram

- **Node Details:**  

  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Initiates the workflow when a Telegram message arrives.  
    - Configuration: Uses a webhook ID to listen to Telegram bot messages.  
    - Inputs: External Telegram message events.  
    - Outputs: Passes Telegram message data downstream.  
    - Edge Cases: Possible webhook or connectivity failure; Telegram API rate limits.  
    - Notes: Requires proper Telegram Bot API credentials and webhook setup.

  - **Telegram**  
    - Type: Telegram  
    - Role: Handles interaction with Telegram API, possibly to fetch or handle audio content from the message.  
    - Configuration: Connected downstream of Telegram Trigger; likely configured to download or process the audio file.  
    - Inputs: Telegram Trigger output.  
    - Outputs: Passes audio file data to transcription.  
    - Edge Cases: File size limits, unsupported audio formats, Telegram API errors.

#### 2.2 AI Transcription

- **Overview:**  
  This block transcribes the received audio message into text using OpenAI’s language model capabilities.

- **Nodes Involved:**  
  - Transcribe Audio  
  - OpenAI

- **Node Details:**  

  - **Transcribe Audio**  
    - Type: OpenAI (LangChain Integration)  
    - Role: Sends audio data for transcription to OpenAI’s audio transcription endpoint.  
    - Configuration: Configured to accept audio input and return transcript text.  
    - Inputs: Audio file from Telegram node.  
    - Outputs: Text transcript of the audio.  
    - Edge Cases: Audio file corruption, transcription errors, API timeouts, limits on audio length or format.

  - **OpenAI**  
    - Type: OpenAI (LangChain Integration)  
    - Role: Processes transcription output further or prepares data for code execution node.  
    - Configuration: Likely set up to refine or clean transcription text or to prepare prompts for next step.  
    - Inputs: Transcribed text from Transcribe Audio node.  
    - Outputs: Passes processed text to the Code node.  
    - Edge Cases: API key issues, response format unexpected, rate limits.

#### 2.3 AI Script Generation

- **Overview:**  
  This block uses the text transcript to generate a structured script outline for podcast notes, using OpenAI’s language generation.

- **Nodes Involved:**  
  - Code  
  - Generate Script Outline

- **Node Details:**  

  - **Code**  
    - Type: Code (JavaScript)  
    - Role: Executes custom JavaScript code to manipulate or prepare data before sending it to the script generation node.  
    - Configuration: Could include formatting, prompt construction, or data validation.  
    - Inputs: Processed transcription text from OpenAI node.  
    - Outputs: Prepared prompt or data to Generate Script Outline node.  
    - Edge Cases: Script errors, invalid data, missing fields.

  - **Generate Script Outline**  
    - Type: OpenAI (LangChain Integration)  
    - Role: Generates a structured outline or script for podcast notes based on input text using GPT or Gemini.  
    - Configuration: Uses a prompt designed to create high-converting or well-structured scripts.  
    - Inputs: Data from Code node.  
    - Outputs: Script outline text.  
    - Edge Cases: Incomplete or irrelevant output, API errors, rate limits.

#### 2.4 Notion Storage

- **Overview:**  
  This block saves both the raw transcript and the generated script outline into Notion for organized storage and later reference.

- **Nodes Involved:**  
  - Save to Notion  
  - Save to Notion1

- **Node Details:**  

  - **Save to Notion**  
    - Type: Notion  
    - Role: Saves the generated script outline into a Notion database or page.  
    - Configuration: Configured with Notion credentials and target database/page.  
    - Inputs: Script outline from Generate Script Outline node.  
    - Outputs: Passes data to the next Notion node or terminates.  
    - Edge Cases: Authentication errors, API limits, missing required fields.

  - **Save to Notion1**  
    - Type: Notion  
    - Role: Saves additional data (likely the raw transcription) into Notion, possibly a different database or page.  
    - Configuration: Similar to Save to Notion but targeting different content or location.  
    - Inputs: Data from Save to Notion node.  
    - Outputs: Workflow end point.  
    - Edge Cases: Same as Save to Notion.

---

### 3. Summary Table

| Node Name            | Node Type                         | Functional Role                    | Input Node(s)         | Output Node(s)           | Sticky Note                                           |
|----------------------|----------------------------------|----------------------------------|-----------------------|--------------------------|------------------------------------------------------|
| Telegram Trigger      | Telegram Trigger                 | Receives Telegram messages       | -                     | Telegram                 |                                                      |
| Telegram             | Telegram                         | Handles Telegram audio message   | Telegram Trigger      | Transcribe Audio          |                                                      |
| Transcribe Audio      | OpenAI (LangChain)               | Transcribes audio to text        | Telegram               | OpenAI                   |                                                      |
| OpenAI               | OpenAI (LangChain)               | Processes transcription output   | Transcribe Audio       | Code                     |                                                      |
| Code                 | Code (JavaScript)                | Prepares data for script outline | OpenAI                 | Generate Script Outline   |                                                      |
| Generate Script Outline | OpenAI (LangChain)             | Generates podcast script outline | Code                   | Save to Notion            |                                                      |
| Save to Notion        | Notion                          | Saves script outline             | Generate Script Outline | Save to Notion1           |                                                      |
| Save to Notion1       | Notion                          | Saves additional data (transcript) | Save to Notion          | -                        |                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Type: Telegram Trigger  
   - Configure with Telegram Bot API credentials.  
   - Set webhook to listen for incoming messages.  
   - This node will start the workflow when a new Telegram message arrives.

2. **Create Telegram Node:**  
   - Type: Telegram  
   - Connect input from Telegram Trigger node.  
   - Configure to fetch or handle incoming audio files from Telegram messages.

3. **Create Transcribe Audio Node:**  
   - Type: OpenAI (LangChain)  
   - Connect input from Telegram node.  
   - Configure to send audio to OpenAI transcription endpoint.  
   - Set the audio file parameter to the file received from Telegram.

4. **Create OpenAI Node:**  
   - Type: OpenAI (LangChain)  
   - Connect input from Transcribe Audio node.  
   - Configure to process or clean transcription results as needed.  
   - Set model and parameters as appropriate for text processing.

5. **Create Code Node:**  
   - Type: Code (JavaScript)  
   - Connect input from OpenAI node.  
   - Write JavaScript code to prepare the transcription text for prompt generation.  
   - Ensure proper error handling and data validation.

6. **Create Generate Script Outline Node:**  
   - Type: OpenAI (LangChain)  
   - Connect input from Code node.  
   - Configure prompt to generate a structured podcast script outline based on input text.  
   - Set model parameters for optimal text generation.

7. **Create Save to Notion Node:**  
   - Type: Notion  
   - Connect input from Generate Script Outline node.  
   - Configure with Notion API credentials.  
   - Set target database or page to save the generated script outline.

8. **Create Save to Notion1 Node:**  
   - Type: Notion  
   - Connect input from Save to Notion node.  
   - Configure similarly to save additional data such as raw transcript into another Notion database/page.

9. **Validate Credentials:**  
   - Ensure Telegram Bot credentials are valid and webhook is active.  
   - Ensure OpenAI API keys are set and have access to transcription and text generation endpoints.  
   - Ensure Notion integration has correct OAuth2 credentials and permissions.

10. **Test Workflow:**  
    - Send an audio message via Telegram to trigger the workflow.  
    - Verify transcription accuracy, script outline quality, and data saving in Notion.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                              |
|----------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| Workflow integrates Telegram, OpenAI (LangChain), and Notion for automated podcast note-taking.                     | Workflow description                          |
| Requires setting up Telegram Bot with proper webhook configured to n8n instance.                                     | Telegram API documentation                    |
| OpenAI nodes utilize LangChain integration for transcription and text generation; ensure API keys cover both uses.  | OpenAI API & LangChain docs                   |
| Notion nodes require OAuth2 credentials with write access to target databases/pages.                                 | Notion API documentation                       |

---

**Disclaimer:** The text provided stems exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected content. All processed data is legal and public.