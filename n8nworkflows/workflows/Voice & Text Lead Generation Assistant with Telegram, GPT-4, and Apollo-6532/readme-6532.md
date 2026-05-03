Voice & Text Lead Generation Assistant with Telegram, GPT-4, and Apollo

https://n8nworkflows.xyz/workflows/voice---text-lead-generation-assistant-with-telegram--gpt-4--and-apollo-6532


# Voice & Text Lead Generation Assistant with Telegram, GPT-4, and Apollo

### 1. Workflow Overview

This workflow implements a **Voice & Text Lead Generation Assistant** integrated with Telegram, GPT-4, and Apollo. It targets users such as cold emailers, growth marketers, solo founders, SDRs, and agencies who want to automate lead scraping and lead research via simple Telegram commands (voice or text). The assistant, personified as "Sam Ovens," can either scrape fresh leads based on user-specified criteria or generate detailed research reports on individual LinkedIn profiles.

The workflow is logically structured into these main blocks:

- **1.1 Input Reception and Type Detection:** Receive Telegram messages, detect whether input is voice or text, and process accordingly.
- **1.2 Audio Transcription (if voice):** Download voice messages and transcribe them to text using OpenAI Whisper.
- **1.3 Conversational AI Agent:** Use a GPT-4-powered agent ("Sam Ovens") to interpret user input, ask clarifying questions, and decide whether to scrape leads or research a lead.
- **1.4 Lead Scraping Tool Call:** Invoke a sub-workflow that scrapes leads based on structured JSON input (location, business, job title).
- **1.5 Lead Research Tool Call:** Invoke a sub-workflow that researches a single lead based on a LinkedIn URL.
- **1.6 Memory Buffer:** Maintain conversational context per user session to enable multi-turn dialogues.
- **1.7 Telegram Response:** Send the AI-generated responses or error messages back to the Telegram user.

The workflow also includes detailed sticky notes with documentation, example outputs, and branding.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Type Detection

- **Overview:** This block captures incoming Telegram messages and determines if the message is a voice note or a text message, directing the flow accordingly.
- **Nodes Involved:** Telegram Trigger, Voice or Text, Download File, Text
- **Node Details:**

  - **Telegram Trigger**
    - *Type:* Telegram Trigger
    - *Role:* Listens for incoming Telegram messages (updates of type "message").
    - *Configuration:* Authenticated with Telegram Bot credentials; webhook set up to receive messages.
    - *Inputs:* Incoming Telegram message update.
    - *Outputs:* Passes full message JSON downstream.
    - *Failures:* Possible webhook misconfiguration, invalid credentials, or Telegram downtime.

  - **Voice or Text (Switch)**
    - *Type:* Switch
    - *Role:* Checks if incoming message contains a voice file or text.
    - *Configuration:* Two rules:
      - Voice: Checks if `message.voice.file_id` exists.
      - Text: Checks if `message.text` exists.
    - *Inputs:* Telegram message JSON.
    - *Outputs:* Routes to Download File if voice; routes to Text node if text.
    - *Failures:* Missing expected fields in JSON could cause routing errors.

  - **Download File**
    - *Type:* Telegram node (file download)
    - *Role:* Downloads the voice file from Telegram servers by `file_id`.
    - *Configuration:* Uses Telegram API credentials and file ID expression from voice message.
    - *Inputs:* Voice message JSON.
    - *Outputs:* Binary audio file data.
    - *Failures:* File ID invalid or expired; Telegram API access errors.

  - **Text**
    - *Type:* Set
    - *Role:* Extracts text from incoming text messages to unify input format.
    - *Configuration:* Assigns `text` attribute from `message.text`.
    - *Inputs:* Telegram message JSON with text.
    - *Outputs:* JSON with `text` property for AI input.
    - *Failures:* Missing or malformed text property.

---

#### 2.2 Audio Transcription

- **Overview:** Converts voice audio files downloaded from Telegram into text using OpenAI’s Whisper API.
- **Nodes Involved:** Transcribe
- **Node Details:**

  - **Transcribe**
    - *Type:* OpenAI audio resource node (transcription)
    - *Role:* Sends the audio file to OpenAI Whisper to obtain a text transcript.
    - *Configuration:* Uses OpenAI API credentials; operation set to "transcribe."
    - *Inputs:* Binary audio data from Download File.
    - *Outputs:* JSON containing transcribed text in the `text` field.
    - *Failures:* Audio file corruption, API quota exceeded, or transcription errors.

---

#### 2.3 Conversational AI Agent

- **Overview:** Processes the unified text input (from transcription or direct text), maintains conversational context, and determines next actions: either lead scraping or lead research. It embodies the persona "Sam Ovens," guiding the user with clarifying questions and calling appropriate tools.
- **Nodes Involved:** Simple Memory, OpenAI Chat Model, Lead Agent
- **Node Details:**

  - **Simple Memory**
    - *Type:* Memory Buffer (windowed)
    - *Role:* Stores and retrieves the last 10 conversational turns per Telegram chat ID to maintain session context.
    - *Configuration:* Session key set as Telegram chat ID; context window length 10.
    - *Inputs:* Incoming text input JSON.
    - *Outputs:* Context-augmented input for AI agent.
    - *Failures:* Session key extraction errors; memory overflow or persistence issues.

  - **OpenAI Chat Model**
    - *Type:* Language Model Node (OpenAI GPT-4)
    - *Role:* Provides GPT-4 model inference for chat responses and reasoning.
    - *Configuration:* Model set to "gpt-4o" variant; uses OpenAI API credentials.
    - *Inputs:* Contextual conversation from Simple Memory.
    - *Outputs:* AI-generated text reply.
    - *Failures:* API quota limits, network issues, or malformed prompts.

  - **Lead Agent**
    - *Type:* LangChain Agent Node
    - *Role:* Central AI agent that interprets user intent, manages tool calls, formats outputs, and error handling.
    - *Configuration:* 
      - System message defines persona, tools ("leadScraping" and "leadResearch"), operational rules, and example dialogues.
      - On error, continues with error output.
      - Uses OpenAI Chat Model as language model.
      - Connected to Simple Memory for conversation history.
      - Calls two sub-workflows as tools for scraping and research.
    - *Inputs:* Text input from transcription or direct text.
    - *Outputs:* Final text output or error message.
    - *Failures:* Expression failures in prompt generation, sub-workflow call failures, or unhandled user input.

---

#### 2.4 Lead Scraping Tool Call

- **Overview:** Invoked by the Lead Agent when user input contains sufficient structured criteria for scraping leads (locations, businesses, job titles). This sub-workflow contacts Apollo or another source to scrape leads and save them to Google Sheets.
- **Nodes Involved:** leadScraping (Tool Workflow node)
- **Node Details:**

  - **leadScraping**
    - *Type:* LangChain Tool Workflow (sub-workflow)
    - *Role:* Scrapes leads based on JSON input with arrays of locations, businesses, and job titles.
    - *Configuration:* 
      - References sub-workflow by ID (IPhiwjZXcxi0UDSf).
      - Requires input JSON structured as:
        ```
        [
          {
            "location": ["location+here", ...],
            "business": ["business+here", ...],
            "job_title": ["job+title+here", ...]
          }
        ]
        ```
      - Description instructs the agent on formatting.
    - *Inputs:* JSON lead scraping query from Lead Agent.
    - *Outputs:* Confirmation message with number of leads added.
    - *Failures:* Input JSON formatting errors, API failures, Google Sheets write errors.

---

#### 2.5 Lead Research Tool Call

- **Overview:** Invoked by the Lead Agent when user input contains a LinkedIn URL for researching a single lead. This sub-workflow generates a professional research report.
- **Nodes Involved:** leadResearch (Tool Workflow node)
- **Node Details:**

  - **leadResearch**
    - *Type:* LangChain Tool Workflow (sub-workflow)
    - *Role:* Researches individual leads by LinkedIn URL.
    - *Configuration:* 
      - References sub-workflow by ID (LAvUyCmqSnNmVrjk).
      - Input JSON format: `{ "linkedinURL": "[URL-HERE]" }`.
      - Input schema includes a `query` string for flexible input mapping.
    - *Inputs:* LinkedIn URL from Lead Agent.
    - *Outputs:* Text research report.
    - *Failures:* Invalid LinkedIn URLs, API errors, malformed input.

---

#### 2.6 Telegram Response

- **Overview:** Sends the final AI-generated response or error message back to the Telegram user.
- **Nodes Involved:** Response, Error Response
- **Node Details:**

  - **Response**
    - *Type:* Telegram node (send message)
    - *Role:* Sends successful responses to the user’s Telegram chat.
    - *Configuration:* Text response from Lead Agent output; chat ID from Telegram Trigger.
    - *Inputs:* AI-generated output text.
    - *Outputs:* None (terminal node).
    - *Failures:* Telegram API errors, chat ID missing.

  - **Error Response**
    - *Type:* Telegram node (send message)
    - *Role:* Sends error messages back to the Telegram chat.
    - *Configuration:* Error text from Lead Agent error output; chat ID from Telegram Trigger.
    - *Inputs:* Error messages.
    - *Outputs:* None.
    - *Failures:* Telegram API errors, chat ID missing.

---

### 3. Summary Table

| Node Name       | Node Type                                | Functional Role                             | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                      |
|-----------------|-----------------------------------------|--------------------------------------------|----------------------|----------------------|-------------------------------------------------------------------------------------------------|
| Telegram Trigger| Telegram Trigger                        | Receives incoming Telegram messages        | -                    | Voice or Text        | Covers message Telegram bot input block                                                        |
| Voice or Text   | Switch                                  | Detects voice or text message type          | Telegram Trigger     | Download File, Text  | Covers message Telegram bot input block                                                        |
| Download File   | Telegram (file download)                 | Downloads voice audio file                   | Voice or Text (Voice) | Transcribe           | Covers message Telegram bot input block                                                        |
| Transcribe      | OpenAI audio transcription              | Transcribes audio to text                    | Download File         | Lead Agent           | Covers message Telegram bot input block                                                        |
| Text            | Set                                     | Extracts text from Telegram text messages   | Voice or Text (Text)  | Lead Agent           | Covers message Telegram bot input block                                                        |
| Simple Memory   | Memory Buffer Window                     | Maintains conversational context            | Lead Agent (input)    | Lead Agent           |                                                                                                 |
| OpenAI Chat Model| Language Model (OpenAI GPT-4)           | Provides GPT-4 chat completions              | Simple Memory         | Lead Agent           |                                                                                                 |
| Lead Agent      | LangChain Agent                         | Conversational AI logic, orchestrates tools | Text, Transcribe, Simple Memory, leadScraping, leadResearch | Response, Error Response | Covers call research or scraping workflow & respond on Telegram                                |
| leadScraping    | LangChain Tool Workflow (Sub-workflow)  | Scrapes leads based on structured query      | Lead Agent (ai_tool)  | Lead Agent (ai_tool) | Covers scraping leads; example outputs shown in sticky notes                                   |
| leadResearch    | LangChain Tool Workflow (Sub-workflow)  | Researches individual leads via LinkedIn URL| Lead Agent (ai_tool)  | Lead Agent (ai_tool) | Covers research report; example outputs shown in sticky notes                                  |
| Response        | Telegram (send message)                  | Sends AI response to Telegram user           | Lead Agent (main)     | -                    |                                                                                                 |
| Error Response  | Telegram (send message)                  | Sends error messages to Telegram user        | Lead Agent (error)    | -                    |                                                                                                 |
| Sticky Note1    | Sticky Note                             | Documentation: Research Report example       | -                    | -                    | ![](https://i.imgur.com/OnzVcDj.png)                                                           |
| Sticky Note     | Sticky Note                             | Documentation: Example Output                 | -                    | -                    | ![](https://i.imgur.com/q3pN5im.png)                                                           |
| Sticky Note2    | Sticky Note                             | Documentation: Scraping Leads                 | -                    | -                    | ![](https://i.imgur.com/qiF5T23.png)                                                           |
| Sticky Note3    | Sticky Note                             | Documentation: Example Output for scraping   | -                    | -                    | ![](https://i.imgur.com/vu7OOqE.png)                                                           |
| Sticky Note4    | Sticky Note                             | Documentation: Message Telegram Bot           | -                    | -                    |                                                                                                 |
| Sticky Note5    | Sticky Note                             | Documentation: Call Research or Scraping workflow & respond on Telegram | -              | -                    |                                                                                                 |
| Sticky Note6    | Sticky Note                             | Author Branding and Contact Info              | -                    | -                    | [https://www.builtbyabdul.com/](https://www.builtbyabdul.com/)                                 |
| Sticky Note7    | Sticky Note                             | Detailed overview, use cases, setup instructions | -                    | -                    | Detailed project overview with setup & customization guidelines                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**
   - Type: Telegram Trigger
   - Set to listen for message updates.
   - Authenticate with your Telegram Bot Token.
   - Position as entry point.

2. **Create Switch Node "Voice or Text"**
   - Type: Switch
   - Add two rules:
     - Voice: Check if `message.voice.file_id` exists.
     - Text: Check if `message.text` exists.
   - Connect Telegram Trigger output to this node.

3. **Create "Download File" Node**
   - Type: Telegram node for file download.
   - Configure to download file by expression: `{{$json.message.voice.file_id}}`.
   - Authenticate with Telegram credentials.
   - Connect "Voice" output of Switch to this node.

4. **Create "Transcribe" Node**
   - Type: OpenAI node (resource: audio, operation: transcribe).
   - Authenticate with OpenAI API credentials.
   - Connect output of Download File to this node.

5. **Create "Text" Node**
   - Type: Set node.
   - Assign variable `text` with expression: `{{$json.message.text}}`.
   - Connect "Text" output of Switch to this node.

6. **Create "Simple Memory" Node**
   - Type: Memory Buffer Window.
   - Session key: `{{$('Telegram Trigger').item.json.message.chat.id}}`.
   - Context window length: 10.
   - Connect outputs of Transcribe and Text nodes to this node (merge or union).

7. **Create "OpenAI Chat Model" Node**
   - Type: Language Model Chat (OpenAI GPT-4).
   - Model: Select GPT-4 variant (e.g., "gpt-4o").
   - Authenticate with OpenAI credentials.
   - Connect Simple Memory output to this node.

8. **Create "Lead Agent" Node (LangChain Agent)**
   - Type: LangChain Agent.
   - Set system prompt with persona "Sam Ovens" including detailed instructions on tools and rules.
   - Reference the OpenAI Chat Model node as the language model.
   - Attach Simple Memory node as AI memory.
   - Add two tools:
     - "leadScraping" tool: references a sub-workflow for lead scraping.
     - "leadResearch" tool: references a sub-workflow for lead research.
   - Enable error continuation.
   - Connect OpenAI Chat Model output and Simple Memory input accordingly.

9. **Create Tool Workflow Nodes**
   - **leadScraping:**
     - Type: LangChain Tool Workflow.
     - Connect to the sub-workflow that scrapes leads.
     - Ensure the input is a structured JSON with location, business, and job_title arrays.
   - **leadResearch:**
     - Type: LangChain Tool Workflow.
     - Connect to the sub-workflow that researches leads by LinkedIn URL.

10. **Connect Lead Agent outputs**
    - Main output to "Response" node.
    - Error output to "Error Response" node.

11. **Create "Response" Node**
    - Type: Telegram node (send message).
    - Text: `{{$json.output}}`.
    - Chat ID: `{{$('Telegram Trigger').item.json.message.chat.id}}`.
    - Authenticate with Telegram credentials.
    - Connect Lead Agent main output to this node.

12. **Create "Error Response" Node**
    - Type: Telegram node (send message).
    - Text: `{{$json.error}}`.
    - Chat ID: `{{$('Telegram Trigger').item.json.message.chat.id}}`.
    - Authenticate with Telegram credentials.
    - Connect Lead Agent error output to this node.

13. **Optional: Add sticky notes for documentation**
    - Add notes describing each functional block, example outputs, and author branding as per original workflow.

14. **Sub-workflows Setup**
    - Import or create the sub-workflow for lead scraping:
      - Inputs: JSON with arrays for location, business, job_title.
      - Outputs: Confirmation string with lead count.
    - Import or create the sub-workflow for lead research:
      - Input: JSON with linkedinURL.
      - Output: Detailed research report text.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                           |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------|
| This workflow personifies the AI agent as "Sam Ovens," a lead generation expert who interacts lightly and humorously with the user. It supports voice input via Telegram voice messages transcribed by OpenAI Whisper, and text input directly. The agent can scrape leads using structured JSON queries and research individual leads via LinkedIn URLs.                                                                                                                                                          | Branding and persona design                               |
| Author: Abdul, who builds growth systems for consultants & agencies. Contact: [https://www.builtbyabdul.com/](https://www.builtbyabdul.com/), Email: builtbyabdul@gmail.com.                                                                                                                                                                                                                                                                                                                                              | Author branding and contact info                          |
| Setup requires Telegram Bot Token, OpenAI API Key (for both chat and Whisper transcription), and Apollo API or equivalent for lead scraping. The workflow assumes sub-workflows for scraping and research are available and correctly configured.                                                                                                                                                                                                                                                                         | Setup instructions                                       |
| Example outputs and usage scenarios are documented with images linked in sticky notes: Research report example ([imgur link](https://i.imgur.com/OnzVcDj.png)), scraping leads example ([imgur link](https://i.imgur.com/qiF5T23.png)), and outputs ([imgur link](https://i.imgur.com/q3pN5im.png), [imgur link](https://i.imgur.com/vu7OOqE.png)).                                                                                                                                                                            | Visual references in sticky notes                         |
| Customization suggestions include replacing Apollo with other scraping APIs, adding CRM integrations (Airtable, HubSpot, Notion), scheduling auto-scrape jobs, PDF report generation, or modifying the agent's persona and tone.                                                                                                                                                                                                                                                                                           | Customization ideas                                      |
| The workflow includes error handling and returns error messages to the user in Telegram to maintain robust user experience.                                                                                                                                                                                                                                                                                                                                                                                              | Error handling design                                    |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.