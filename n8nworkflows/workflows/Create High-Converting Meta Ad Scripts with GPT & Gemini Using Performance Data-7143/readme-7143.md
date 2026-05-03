Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data

https://n8nworkflows.xyz/workflows/create-high-converting-meta-ad-scripts-with-gpt---gemini-using-performance-data-7143


# Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data

### 1. Workflow Overview

This workflow, titled **"Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data"**, is designed to automate the transcription of podcast audio received via Telegram, process the transcription with OpenAI GPT models to generate structured script outlines, and save the results into Notion databases. The workflow targets podcast content creators and marketers who want to efficiently convert spoken content into actionable, structured script outlines for Meta ads or other marketing scripts.

The workflow logic is organized into the following blocks:

- **1.1 Input Reception:** Receives audio input via Telegram messages.
- **1.2 Audio Transcription:** Converts the audio message into text using OpenAI transcription models.
- **1.3 AI Text Processing:** Uses multiple OpenAI nodes and a Code node to transform the transcription into a high-level script outline.
- **1.4 Data Persistence:** Saves the generated script outline data into Notion for organization and later use.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block handles receiving user input as audio messages via Telegram. It triggers workflow execution whenever a new Telegram message is received.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Telegram

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Trigger node for Telegram messages  
    - Configuration: Listens to incoming Telegram messages to initiate the workflow.  
    - Expressions: None critical; uses webhook to receive updates.  
    - Input: External Telegram messages (audio or other media).  
    - Output: Passes message data to the Telegram node.  
    - Edge Cases: Telegram API limits, malformed messages, unsupported media types.  
    - Version: v1 (standard n8n Telegram Trigger).

  - **Telegram**  
    - Type: Telegram node (standard)  
    - Configuration: Receives the audio content from the Telegram Trigger node and processes it for further transcription.  
    - Expressions: None explicitly defined.  
    - Input: Data from Telegram Trigger.  
    - Output: Passes audio data to the Transcribe Audio node.  
    - Edge Cases: Telegram API errors, missing audio content, network issues.  
    - Version: v1.2.

---

#### 1.2 Audio Transcription

- **Overview:**  
  This block transcribes the audio received from Telegram into text using OpenAI’s language model specialized for transcription.

- **Nodes Involved:**  
  - Transcribe Audio  
  - OpenAI

- **Node Details:**

  - **Transcribe Audio**  
    - Type: OpenAI node (LangChain implementation)  
    - Configuration: Uses OpenAI’s transcription capabilities to convert audio files into text.  
    - Expressions: Likely uses input audio binary data from Telegram node.  
    - Input: Audio data from Telegram node.  
    - Output: Text transcription to be sent to the next OpenAI node for further processing.  
    - Edge Cases: Audio file format incompatibilities, transcription inaccuracies, API rate limiting.  
    - Version: v1.

  - **OpenAI**  
    - Type: OpenAI node (LangChain implementation)  
    - Configuration: Processes the raw transcription text, possibly cleaning or formatting it before the next step.  
    - Expressions: Uses transcription text from previous node.  
    - Input: Transcription text from Transcribe Audio.  
    - Output: Preprocessed text to the Code node.  
    - Edge Cases: API failures, invalid input text, prompt errors.  
    - Version: v1.8.

---

#### 1.3 AI Text Processing

- **Overview:**  
  This block refines the transcription text into a structured script outline using a combination of code transformations and advanced OpenAI prompts.

- **Nodes Involved:**  
  - Code  
  - Generate Script Outline

- **Node Details:**

  - **Code**  
    - Type: Code node (JavaScript)  
    - Configuration: Executes custom JavaScript to transform or prepare the text data for the final AI prompt.  
    - Expressions: Likely manipulates or restructures the text data from OpenAI node.  
    - Input: Text data from OpenAI node.  
    - Output: Prepared input for the Generate Script Outline node.  
    - Edge Cases: Script errors, unexpected input formats, runtime exceptions.  
    - Version: v2.

  - **Generate Script Outline**  
    - Type: OpenAI node (LangChain implementation)  
    - Configuration: Uses GPT or Gemini models to generate a detailed and high-converting script outline based on the processed transcription text.  
    - Expressions: Prompts designed to convert podcast transcription into Meta ad scripts or marketing outlines.  
    - Input: Prepared text from Code node.  
    - Output: Structured script outline data to be saved in Notion.  
    - Edge Cases: Prompt misinterpretation, API errors, large input size exceeding token limits.  
    - Version: v1.8.

---

#### 1.4 Data Persistence

- **Overview:**  
  This block saves the generated script outline into Notion databases to organize and store the marketing scripts.

- **Nodes Involved:**  
  - Save to Notion  
  - Save to Notion1

- **Node Details:**

  - **Save to Notion**  
    - Type: Notion node  
    - Configuration: Saves the structured script outline into a Notion database (likely the first step or intermediate save).  
    - Expressions: Maps data fields from Generate Script Outline output to Notion properties.  
    - Input: Script outline data from Generate Script Outline node.  
    - Output: Passes data to Save to Notion1 node.  
    - Edge Cases: Notion API authentication errors, rate limits, missing database or property fields.  
    - Version: v2.2.

  - **Save to Notion1**  
    - Type: Notion node  
    - Configuration: Final save step or additional data enrichment in the same or another Notion database.  
    - Expressions: Continues or supplements the previous Notion save operation.  
    - Input: Output from Save to Notion node.  
    - Output: End of workflow (no further nodes connected).  
    - Edge Cases: Same as above.  
    - Version: v2.2.

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role               | Input Node(s)      | Output Node(s)           | Sticky Note                                      |
|---------------------|----------------------------------|------------------------------|--------------------|--------------------------|-------------------------------------------------|
| Telegram Trigger     | n8n-nodes-base.telegramTrigger   | Entry point: receive messages | -                  | Telegram                 |                                                 |
| Telegram            | n8n-nodes-base.telegram          | Process received Telegram msg | Telegram Trigger   | Transcribe Audio         |                                                 |
| Transcribe Audio     | @n8n/n8n-nodes-langchain.openAi | Transcribe audio to text      | Telegram            | OpenAI                   |                                                 |
| OpenAI              | @n8n/n8n-nodes-langchain.openAi | Preprocess transcription text | Transcribe Audio    | Code                     |                                                 |
| Code                | n8n-nodes-base.code              | Prepare data for script gen   | OpenAI              | Generate Script Outline  |                                                 |
| Generate Script Outline | @n8n/n8n-nodes-langchain.openAi | Generate script outline       | Code                | Save to Notion           |                                                 |
| Save to Notion      | n8n-nodes-base.notion            | Save outline to Notion DB     | Generate Script Outline | Save to Notion1          |                                                 |
| Save to Notion1     | n8n-nodes-base.notion            | Final save in Notion DB       | Save to Notion      | -                        |                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Node Type: Telegram Trigger  
   - Configure with the Telegram bot’s webhook to listen for new messages (audio).  
   - No additional parameters needed.

2. **Create Telegram Node**  
   - Node Type: Telegram  
   - Connect input from Telegram Trigger.  
   - Configure credentials with Telegram Bot API token.  
   - Purpose: Receive audio message content for transcription.

3. **Create Transcribe Audio Node**  
   - Node Type: OpenAI LangChain node  
   - Connect input from Telegram node.  
   - Configure to use OpenAI Whisper or equivalent transcription model.  
   - Set audio input parameter to receive binary data from Telegram.

4. **Create OpenAI Node (Text Preprocessing)**  
   - Node Type: OpenAI LangChain node  
   - Connect input from Transcribe Audio node.  
   - Configure prompt to clean or format transcription text.  
   - Use OpenAI API credentials with necessary scopes.

5. **Create Code Node**  
   - Node Type: Code (JavaScript)  
   - Connect input from OpenAI node.  
   - Write JavaScript code to manipulate or prepare transcription text into a suitable prompt or format for script generation.

6. **Create Generate Script Outline Node**  
   - Node Type: OpenAI LangChain node  
   - Connect input from Code node.  
   - Configure prompt to generate a high-converting Meta ad script outline from the prepared text.  
   - Use OpenAI or Gemini model credentials.

7. **Create Save to Notion Node**  
   - Node Type: Notion  
   - Connect input from Generate Script Outline node.  
   - Configure credentials to Notion API.  
   - Map output fields from the script outline to respective Notion database properties.

8. **Create Save to Notion1 Node**  
   - Node Type: Notion  
   - Connect input from Save to Notion node.  
   - Configure similarly for either the same or a different Notion database for final data storage.

9. **Set Execution Order and Test Workflow**  
   - Verify all nodes are connected as per above.  
   - Test with Telegram audio messages to confirm transcription, script generation, and saving to Notion.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                            |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Workflow tags include: ai, podcast, notion—indicating focus on AI transcription and note-taking. | Workflow metadata                                           |
| This workflow uses LangChain OpenAI nodes, requiring valid OpenAI API credentials.               | OpenAI documentation: https://platform.openai.com/docs    |
| Telegram nodes require a bot token and webhook setup.                                            | Telegram Bot API: https://core.telegram.org/bots/api       |
| Notion nodes require integration with a Notion workspace and database configured for script data.| Notion API docs: https://developers.notion.com/            |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.