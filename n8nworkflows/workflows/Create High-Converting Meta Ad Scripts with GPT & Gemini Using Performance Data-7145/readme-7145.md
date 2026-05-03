Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data

https://n8nworkflows.xyz/workflows/create-high-converting-meta-ad-scripts-with-gpt---gemini-using-performance-data-7145


# Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data

### 1. Workflow Overview

This workflow, titled **"Create High-Converting Meta Ad Scripts with GPT & Gemini Using Performance Data"**, is designed to automate the transcription and summarization of podcast audio received via Telegram, and to save the generated content into Notion for further use. The core use case is to convert audio podcast inputs into structured script outlines using AI language models, facilitating content creation and management.

The workflow is logically divided into the following functional blocks:

- **1.1 Input Reception:** Receiving audio input from Telegram messages.
- **1.2 Audio Transcription:** Using AI to transcribe audio into text.
- **1.3 Text Processing & Script Generation:** Applying AI to summarize or generate script outlines from transcribed text.
- **1.4 Data Storage:** Saving the generated script outlines and related data into Notion databases.
- **1.5 Messaging Output:** Sending back confirmations or messages through Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block listens for incoming Telegram messages (likely containing podcast audio or commands) and triggers the workflow.

**Nodes Involved:**  
- Telegram Trigger  
- Telegram  

**Node Details:**  

- **Telegram Trigger**  
  - Type: `telegramTrigger`  
  - Role: Listens for new Telegram messages or commands to start the workflow.  
  - Configuration: Uses a webhook ID to receive incoming Telegram updates.  
  - Inputs: External Telegram client messages.  
  - Outputs: Passes message data to the Telegram node.  
  - Potential Failures: Webhook misconfiguration, Telegram API downtime, or auth token expiration.

- **Telegram**  
  - Type: `telegram`  
  - Role: Processes or sends messages back to Telegram users.  
  - Configuration: Linked to Telegram Trigger node; likely configured to send confirmations or receive audio files.  
  - Inputs: Output from Telegram Trigger.  
  - Outputs: Sends data to the next node, `Transcribe Audio`.  
  - Potential Failures: Authentication errors, message format errors, or rate limits.

---

#### 2.2 Audio Transcription

**Overview:**  
This block uses an AI language model to transcribe the received podcast audio into text format.

**Nodes Involved:**  
- Transcribe Audio  
- OpenAI  

**Node Details:**  

- **Transcribe Audio**  
  - Type: `@n8n/n8n-nodes-langchain.openAi` (custom Langchain OpenAI node)  
  - Role: Converts audio data into text transcription using OpenAI's speech-to-text capabilities or similar.  
  - Configuration: Likely configured with appropriate OpenAI credentials and model selection for transcription.  
  - Inputs: Audio data from Telegram node.  
  - Outputs: Transcribed text to the OpenAI node for further processing.  
  - Potential Failures: Audio format issues, transcription model errors, API rate limits.

- **OpenAI**  
  - Type: `@n8n/n8n-nodes-langchain.openAi`  
  - Role: Processes transcribed text for cleanup or preliminary NLP tasks before scripting.  
  - Configuration: Uses OpenAI API key and possibly specific prompt templates.  
  - Inputs: Transcribed text from "Transcribe Audio".  
  - Outputs: Passes processed text to the Code node.  
  - Potential Failures: API errors, prompt misconfigurations.

---

#### 2.3 Text Processing & Script Generation

**Overview:**  
This block generates a structured script outline from the processed transcription to facilitate content creation.

**Nodes Involved:**  
- Code  
- Generate Script Outline  

**Node Details:**  

- **Code**  
  - Type: `code` (JavaScript/TypeScript execution node)  
  - Role: Custom scripting to manipulate or format the OpenAI output before final script outline generation.  
  - Configuration: Contains custom logic to prepare data or enrich context for the next AI call.  
  - Inputs: Output from OpenAI node.  
  - Outputs: Passes refined data to "Generate Script Outline".  
  - Potential Failures: Script errors, undefined variables, or data format mismatches.

- **Generate Script Outline**  
  - Type: `@n8n/n8n-nodes-langchain.openAi`  
  - Role: Uses AI to generate a high-level script outline from the processed transcription data.  
  - Configuration: Uses specific prompt engineering to create ad or podcast script outlines.  
  - Inputs: Refined data from Code node.  
  - Outputs: Sends generated script outline to Notion saving node.  
  - Potential Failures: API errors, incomplete data, or prompt failure.

---

#### 2.4 Data Storage

**Overview:**  
This block saves the generated script outlines into Notion databases for structured storage and future reference.

**Nodes Involved:**  
- Save to Notion  
- Save to Notion1  

**Node Details:**  

- **Save to Notion**  
  - Type: `notion`  
  - Role: Inserts or updates data in Notion with the generated script outline.  
  - Configuration: Configured with Notion integration credentials and target database/page.  
  - Inputs: Script outline from "Generate Script Outline".  
  - Outputs: Passes control to "Save to Notion1".  
  - Potential Failures: API limits, permission errors, or data validation issues.

- **Save to Notion1**  
  - Type: `notion`  
  - Role: Possibly a secondary or follow-up saving step (e.g., saving metadata or confirmations).  
  - Configuration: Similar to "Save to Notion" but potentially different database or page configuration.  
  - Inputs: From "Save to Notion".  
  - Outputs: Terminal node with no further outputs.  
  - Potential Failures: Same as above.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                  | Input Node(s)         | Output Node(s)        | Sticky Note                            |
|-----------------------|----------------------------------|--------------------------------|-----------------------|-----------------------|--------------------------------------|
| Telegram Trigger       | telegramTrigger                  | Receive Telegram messages       | -                     | Telegram              |                                      |
| Telegram              | telegram                        | Process/send Telegram messages  | Telegram Trigger      | Transcribe Audio      |                                      |
| Transcribe Audio       | @n8n/n8n-nodes-langchain.openAi| Transcribe audio to text        | Telegram              | OpenAI                |                                      |
| OpenAI                | @n8n/n8n-nodes-langchain.openAi| Process transcription text      | Transcribe Audio      | Code                  |                                      |
| Code                  | code                            | Custom processing / formatting  | OpenAI                | Generate Script Outline|                                      |
| Generate Script Outline| @n8n/n8n-nodes-langchain.openAi| Generate script outline         | Code                  | Save to Notion        |                                      |
| Save to Notion        | notion                          | Save script outline to Notion   | Generate Script Outline| Save to Notion1       |                                      |
| Save to Notion1       | notion                          | Secondary save to Notion        | Save to Notion        | -                     |                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: `telegramTrigger`  
   - Configure with your Telegram bot credentials and webhook URL.  
   - Purpose: To listen for incoming Telegram messages that start the workflow.

2. **Add Telegram node**  
   - Type: `telegram`  
   - Connect input from Telegram Trigger.  
   - Configure to process incoming messages or send replies as needed (bot token required).

3. **Add Transcribe Audio node**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Connect input from Telegram node output.  
   - Configure OpenAI credentials with transcription model or equivalent.  
   - Set parameters to accept audio input and return text transcription.

4. **Add OpenAI node**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Connect input from Transcribe Audio node.  
   - Configure with OpenAI API credentials.  
   - Set prompt to process or clean transcription text.

5. **Add Code node**  
   - Type: `code`  
   - Connect input from OpenAI node.  
   - Insert JavaScript code that formats or enriches the text for script generation.

6. **Add Generate Script Outline node**  
   - Type: `@n8n/n8n-nodes-langchain.openAi`  
   - Connect input from Code node.  
   - Configure with OpenAI credentials and prompt engineered to produce script outlines.

7. **Add Save to Notion node**  
   - Type: `notion`  
   - Connect input from Generate Script Outline.  
   - Configure Notion API credentials and select target database/page for storing outlines.

8. **Add Save to Notion1 node**  
   - Type: `notion`  
   - Connect input from Save to Notion.  
   - Configure similarly or for secondary data storage.

9. **Set execution order and activate the workflow**  
   - Ensure nodes are connected as per above order.  
   - Test each step for authentication and data correctness.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                           |
|------------------------------------------------------------------------------|-----------------------------------------------------------|
| The workflow tags include `ai`, `podcast`, and `notion` indicating use cases related to AI-based podcast content processing and Notion integration. | Tag context                                               |
| The Langchain OpenAI nodes require valid OpenAI API keys and specific model access rights for transcription and text generation tasks. | OpenAI docs: https://platform.openai.com/docs             |
| Telegram integration needs a Telegram bot token and properly set webhooks for real-time message processing. | Telegram Bot API: https://core.telegram.org/bots/api      |
| Notion integration requires an integration token and database/page ID with permissions to create or update content. | Notion API: https://developers.notion.com/docs             |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.