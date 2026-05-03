Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data

https://n8nworkflows.xyz/workflows/create-high-converting-meta-ad-scripts-with-gpt---gemini-using-performance-data-7155


# Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data

### 1. Workflow Overview

This workflow, titled **"Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data"**, is designed to automate the process of transcribing audio content received from Telegram, generating structured scripts based on the transcriptions, and saving the results into Notion for further use. The workflow is well suited for podcast creators, marketers, or content managers who want to efficiently convert spoken content into actionable notes and scripts leveraging AI capabilities.

The workflow logic is divided into four main blocks:

- **1.1 Input Reception:** Captures incoming audio messages from Telegram via a webhook trigger.
- **1.2 Audio Transcription:** Uses OpenAI’s language model (accessed through n8n’s LangChain integration) to transcribe the audio content into text.
- **1.3 Script Generation:** Processes the transcription with additional AI prompting to create structured script outlines.
- **1.4 Data Storage:** Saves both the generated script outlines and additional processed notes into Notion databases for organization and retrieval.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** This block listens for incoming audio messages via Telegram and triggers the workflow to start processing as soon as a message arrives.
- **Nodes Involved:**  
  - Telegram Trigger  
  - Telegram

- **Node Details:**

  1. **Telegram Trigger**  
     - *Type:* Telegram Trigger  
     - *Role:* Entry point for the workflow, listens for new messages on a configured Telegram bot webhook.  
     - *Configuration:* Uses a webhook ID to receive message updates from Telegram.  
     - *Key Expressions:* None specified explicitly; defaults to capturing all message payloads.  
     - *Input:* External Telegram webhook.  
     - *Output:* Passes the message data to the Telegram node.  
     - *Edge Cases:* Potential webhook downtime, Telegram API rate limiting, missing or malformed message data.

  2. **Telegram**  
     - *Type:* Telegram node  
     - *Role:* Retrieves or processes the incoming Telegram message data, potentially extracting audio file information for transcription.  
     - *Configuration:* Linked to the same webhook as the trigger, set up with credentials to interact with Telegram API.  
     - *Input:* Connected from Telegram Trigger output.  
     - *Output:* Sends audio file data or message content to the transcription step.  
     - *Edge Cases:* API authentication failures, audio file missing or unsupported format.

#### 2.2 Audio Transcription

- **Overview:** Receives audio data from Telegram and transcribes it into text using an OpenAI model integrated via LangChain.
- **Nodes Involved:**  
  - Transcribe Audio  
  - OpenAI

- **Node Details:**

  1. **Transcribe Audio**  
     - *Type:* OpenAI node (LangChain integration)  
     - *Role:* Converts audio input into text using AI transcription capabilities.  
     - *Configuration:* Likely configured to use an OpenAI speech-to-text model or a prompt-based transcription approach.  
     - *Input:* Audio data from Telegram node.  
     - *Output:* Transcribed text passed to the OpenAI node for further processing.  
     - *Edge Cases:* Audio quality issues, transcription inaccuracies, API timeouts.

  2. **OpenAI**  
     - *Type:* OpenAI node (LangChain integration)  
     - *Role:* Potentially processes or refines transcription text before coding or scripting steps.  
     - *Configuration:* Uses GPT or similar model with prompt templates tailored for transcription refinement.  
     - *Input:* Output from Transcribe Audio node.  
     - *Output:* Text data passed to the Code node.  
     - *Edge Cases:* Rate limits, prompt execution errors.

#### 2.3 Script Generation

- **Overview:** Takes refined transcription text and generates a structured script outline for meta advertisement or podcast notes.
- **Nodes Involved:**  
  - Code  
  - Generate Script Outline

- **Node Details:**

  1. **Code**  
     - *Type:* Code node (JavaScript execution)  
     - *Role:* Processes or formats the text data programmatically to prepare it for AI script generation.  
     - *Configuration:* Custom JavaScript logic likely to clean, parse, or enrich transcription text.  
     - *Input:* Text from OpenAI node.  
     - *Output:* Processed data sent to Generate Script Outline.  
     - *Edge Cases:* Script errors, unexpected data formats.

  2. **Generate Script Outline**  
     - *Type:* OpenAI node (LangChain integration)  
     - *Role:* Generates a high-level script outline from the processed transcription using GPT or Gemini AI models.  
     - *Configuration:* Configured with prompts designed to create engaging, high-converting ad scripts or structured notes.  
     - *Input:* Data from Code node.  
     - *Output:* Script outline data sent to Notion storage.  
     - *Edge Cases:* AI model response failures, prompt misunderstandings.

#### 2.4 Data Storage

- **Overview:** Saves the generated script outlines and notes into Notion for persistent storage and future reference.
- **Nodes Involved:**  
  - Save to Notion  
  - Save to Notion1

- **Node Details:**

  1. **Save to Notion**  
     - *Type:* Notion node  
     - *Role:* Stores the generated script outline into a specific Notion database or page.  
     - *Configuration:* Uses Notion API credentials with access to the target database/page.  
     - *Input:* Script outline from Generate Script Outline node.  
     - *Output:* Passes data to Save to Notion1 for further storage or chaining.  
     - *Edge Cases:* API authentication issues, database schema mismatches.

  2. **Save to Notion1**  
     - *Type:* Notion node  
     - *Role:* Further saves or updates additional related data in Notion, possibly detailed notes or metadata.  
     - *Configuration:* Similar Notion API setup as the previous node, targeting another database or page.  
     - *Input:* From Save to Notion node.  
     - *Output:* Terminal node (no further outputs).  
     - *Edge Cases:* Same as Save to Notion, plus possibility of partial data writes.

---

### 3. Summary Table

| Node Name            | Node Type                     | Functional Role                       | Input Node(s)        | Output Node(s)         | Sticky Note                              |
|----------------------|-------------------------------|-------------------------------------|----------------------|------------------------|-----------------------------------------|
| Telegram Trigger      | Telegram Trigger               | Trigger workflow on Telegram message| None                 | Telegram               |                                         |
| Telegram             | Telegram                      | Retrieve/process incoming message    | Telegram Trigger     | Transcribe Audio       |                                         |
| Transcribe Audio      | OpenAI (LangChain)            | Transcribe audio to text             | Telegram             | OpenAI                 |                                         |
| OpenAI                | OpenAI (LangChain)            | Refine/process transcription text   | Transcribe Audio      | Code                   |                                         |
| Code                  | Code (JavaScript)             | Process text programmatically        | OpenAI                | Generate Script Outline |                                         |
| Generate Script Outline| OpenAI (LangChain)           | Generate structured script outline   | Code                  | Save to Notion         |                                         |
| Save to Notion        | Notion                       | Save script outline to Notion        | Generate Script Outline| Save to Notion1        |                                         |
| Save to Notion1       | Notion                       | Save additional notes to Notion      | Save to Notion        | None                   |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Node type: Telegram Trigger  
   - Configuration: Set up a webhook to receive Telegram messages. Use your Telegram bot credentials.  
   - No input connections.  
   - Output connected to the Telegram node.

2. **Create Telegram Node**
   - Node type: Telegram  
   - Configure with Telegram bot credentials to access messages and media.  
   - Connect input from Telegram Trigger output.  
   - Output connected to Transcribe Audio node.

3. **Create Transcribe Audio Node**
   - Node type: OpenAI (LangChain integration)  
   - Configure with OpenAI API credentials (ensure speech-to-text or transcription model enabled).  
   - Connect input from Telegram node.  
   - Output connected to OpenAI node.

4. **Create OpenAI Node for Text Processing**
   - Node type: OpenAI (LangChain integration)  
   - Configure with OpenAI API credentials.  
   - Set prompt or model to refine transcription text.  
   - Connect input from Transcribe Audio node.  
   - Output connected to Code node.

5. **Create Code Node**
   - Node type: Code (JavaScript)  
   - Write custom JavaScript to prepare/refine transcription text for script generation (e.g., cleaning, formatting).  
   - Connect input from OpenAI node.  
   - Output connected to Generate Script Outline node.

6. **Create Generate Script Outline Node**
   - Node type: OpenAI (LangChain integration)  
   - Configure with OpenAI API credentials.  
   - Provide prompt templates to generate a structured, high-converting meta ad script outline from text input.  
   - Connect input from Code node.  
   - Output connected to Save to Notion node.

7. **Create Save to Notion Node**
   - Node type: Notion  
   - Configure with Notion API credentials.  
   - Select database or page to save the script outline.  
   - Connect input from Generate Script Outline node.  
   - Output connected to Save to Notion1 node.

8. **Create Save to Notion1 Node**
   - Node type: Notion  
   - Configure similarly with Notion API credentials.  
   - Select target database or page for extra notes or metadata.  
   - Connect input from Save to Notion node.  
   - No output connections (terminal node).

9. **Set Workflow Execution Settings**
   - Ensure execution order is set to ‘v1’ (sequential).  
   - Verify all credentials (Telegram, OpenAI, Notion) are valid and tested.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                        |
|-----------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| This workflow leverages n8n’s LangChain OpenAI nodes to integrate GPT and potentially Gemini AI models for transcription and script generation. | n8n LangChain OpenAI Node Documentation                              |
| Ensure Telegram bot webhook is properly configured and publicly accessible for triggers to work.| Telegram Bot API Documentation                                        |
| Notion API requires integration token with access to the specific database or pages used.      | Notion API Documentation                                              |
| Consider handling large audio files with appropriate size limits and timeouts in OpenAI API.   | OpenAI API Rate Limits and Best Practices                             |
| Workflow designed for podcast note-taking but adaptable to other audio-to-text script needs.   | Workflow tags: ai, podcast, notion                                    |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.