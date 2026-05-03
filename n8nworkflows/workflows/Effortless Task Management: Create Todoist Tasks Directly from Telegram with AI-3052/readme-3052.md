Effortless Task Management: Create Todoist Tasks Directly from Telegram with AI

https://n8nworkflows.xyz/workflows/effortless-task-management--create-todoist-tasks-directly-from-telegram-with-ai-3052


# Effortless Task Management: Create Todoist Tasks Directly from Telegram with AI

### 1. Workflow Overview

This workflow enables effortless task management by creating Todoist tasks directly from messages sent to a Telegram bot. It supports both voice and text messages, leveraging AI to intelligently analyze and break down user input into actionable sub-tasks with priorities. The workflow automates task creation in Todoist and sends a confirmation back to the user via Telegram.

**Target Use Cases:**  
- Busy professionals capturing tasks on the go  
- Students managing assignments and projects  
- Anyone wanting AI-assisted task breakdown and management  

**Logical Blocks:**  
- **1.1 Input Reception:** Receiving and routing Telegram messages (voice or text)  
- **1.2 Voice Message Processing:** Fetching and transcribing voice messages to text  
- **1.3 Text Preparation:** Preparing text input for AI processing  
- **1.4 AI Task Analysis:** Using OpenAI GPT to decompose input into structured sub-tasks  
- **1.5 Task Creation:** Creating tasks in Todoist based on AI output  
- **1.6 User Notification:** Sending confirmation messages back to Telegram users  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for incoming Telegram messages and routes them based on whether they contain voice or text content.

**Nodes Involved:**  
- Receive Telegram Messages  
- Voice or Text? (Switch)  
- Sticky Note2 (commentary)

**Node Details:**  

- **Receive Telegram Messages**  
  - Type: Telegram Trigger  
  - Role: Listens for new messages sent to the Telegram bot  
  - Configuration: Listens for "message" updates only; uses Telegram API credentials  
  - Inputs: External webhook trigger from Telegram  
  - Outputs: JSON containing message data  
  - Edge Cases: Missing or malformed messages, Telegram API downtime  
  - Notes: Entry point of the workflow  

- **Voice or Text?**  
  - Type: Switch  
  - Role: Routes messages into three outputs: Audio (voice), Text, or Error  
  - Configuration:  
    - Audio branch if `message.voice.file_id` exists  
    - Text branch if `message.text` exists  
    - Error branch if `error` field exists  
  - Inputs: Output from Telegram Trigger  
  - Outputs: Three branches for downstream processing  
  - Edge Cases: Messages without voice or text, unexpected message formats  

- **Sticky Note2**  
  - Type: Sticky Note  
  - Content: "This workflow listens for incoming voice or text messages from Telegram users."  
  - Role: Documentation  

---

#### 1.2 Voice Message Processing

**Overview:**  
Handles voice messages by fetching the audio file from Telegram and transcribing it into text using OpenAI's Whisper API.

**Nodes Involved:**  
- Fetch Voice Message  
- Transcribe Voice to Text  
- Sticky Note3 (commentary)

**Node Details:**  

- **Fetch Voice Message**  
  - Type: Telegram node  
  - Role: Downloads the voice message file using Telegram API  
  - Configuration: Uses `message.voice.file_id` from input JSON to fetch the file  
  - Inputs: Audio branch from Switch node  
  - Outputs: Binary audio file data  
  - Edge Cases: Invalid file ID, Telegram API errors, file download failures  

- **Transcribe Voice to Text**  
  - Type: OpenAI node (audio translate operation)  
  - Role: Transcribes audio to text using OpenAI Whisper API  
  - Configuration: Resource set to "audio", operation "translate"  
  - Inputs: Binary audio data from Fetch Voice Message  
  - Outputs: JSON with transcribed text  
  - Edge Cases: API rate limits, transcription errors, unsupported audio formats  

- **Sticky Note3**  
  - Type: Sticky Note  
  - Content: "Voice messages are fetched from Telegram and transcribed into text using OpenAI's Whisper API."  

---

#### 1.3 Text Preparation

**Overview:**  
Prepares the text input (either from direct text messages or transcribed voice messages) for AI processing by assigning it to a unified field.

**Nodes Involved:**  
- Prepare for LLM

**Node Details:**  

- **Prepare for LLM**  
  - Type: Set node  
  - Role: Assigns the text content to a `text` field for consistent downstream processing  
  - Configuration: Sets `text` to `{{$json.message.text}}` (for text messages) or transcribed text (for voice)  
  - Inputs: Text branch from Switch or output from Transcribe Voice to Text  
  - Outputs: JSON with unified `text` field  
  - Edge Cases: Missing text field, empty messages  

---

#### 1.4 AI Task Analysis

**Overview:**  
Uses an OpenAI GPT model to analyze the input text and break it down into structured sub-tasks formatted for Todoist.

**Nodes Involved:**  
- OpenAI Chat Model  
- Basic LLM Chain  
- Extract Tasks (Structured Output Parser)  
- Sticky Note4 (commentary)

**Node Details:**  

- **OpenAI Chat Model**  
  - Type: Langchain OpenAI Chat Model node  
  - Role: Provides the GPT-4o-mini model for language understanding  
  - Configuration: Model set to "gpt-4o-mini" with OpenAI API credentials  
  - Inputs: Connected as language model for Basic LLM Chain  
  - Outputs: Language model responses  
  - Edge Cases: API key issues, rate limits, model unavailability  

- **Basic LLM Chain**  
  - Type: Langchain LLM Chain node  
  - Role: Sends the prepared text to the AI with a detailed prompt to decompose tasks  
  - Configuration:  
    - Text input: `={{ $json.text }}`  
    - Prompt: Detailed system prompt instructing AI to break down tasks into JSON objects with `content` and `priority` fields  
  - Inputs: Text from Prepare for LLM or transcribed text  
  - Outputs: AI-generated JSON list of sub-tasks  
  - Edge Cases: Invalid JSON output, ambiguous user input, prompt failures  

- **Extract Tasks**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses the AI response to extract structured JSON sub-tasks  
  - Configuration: Example JSON schema provided for validation  
  - Inputs: AI output from Basic LLM Chain  
  - Outputs: Parsed JSON array of tasks with `content` and `priority`  
  - Edge Cases: Parsing errors if AI output is malformed  

- **Sticky Note4**  
  - Type: Sticky Note  
  - Content: "The LLM (OpenAI Chat Model) analyzes the text and breaks it down into tasks and sub-tasks, formatted for Todoist."  

---

#### 1.5 Task Creation

**Overview:**  
Creates tasks in the user's Todoist project based on the AI-generated sub-tasks.

**Nodes Involved:**  
- Create Todoist Tasks

**Node Details:**  

- **Create Todoist Tasks**  
  - Type: Todoist node  
  - Role: Creates individual tasks in Todoist using AI output  
  - Configuration:  
    - Content: `={{ $json.output.content }}` (task description)  
    - Priority: `={{ $json.output.priority }}` (1-4 scale)  
    - Project: Fixed project ID `"2349786654"` (can be customized)  
  - Inputs: Parsed sub-tasks from Extract Tasks  
  - Outputs: JSON with created task details including task URL  
  - Credentials: Todoist API credentials required  
  - Edge Cases: API errors, invalid project ID, rate limits  

---

#### 1.6 User Notification

**Overview:**  
Sends a confirmation message back to the Telegram user with a link to the created Todoist task.

**Nodes Involved:**  
- Send Confirmation

**Node Details:**  

- **Send Confirmation**  
  - Type: Telegram node  
  - Role: Sends a message to the Telegram chat confirming task creation  
  - Configuration:  
    - Text: Template string including task content and Todoist task URL  
    - Chat ID: Extracted from original Telegram message (`{{$('Receive Telegram Messages').item.json.message.chat.id}}`)  
  - Inputs: Output from Create Todoist Tasks  
  - Outputs: Confirmation message sent to user  
  - Edge Cases: Invalid chat ID, Telegram API errors  

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                                   | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                  |
|-----------------------|----------------------------------|-------------------------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------|
| Receive Telegram Messages | Telegram Trigger                 | Listens for incoming Telegram messages           | (Webhook trigger)            | Voice or Text?             | This workflow listens for incoming voice or text messages from Telegram users.               |
| Voice or Text?         | Switch                           | Routes messages based on type (voice, text, error) | Receive Telegram Messages    | Fetch Voice Message, Prepare for LLM | This workflow listens for incoming voice or text messages from Telegram users.               |
| Fetch Voice Message    | Telegram                         | Downloads voice message audio file from Telegram | Voice or Text? (Audio branch) | Transcribe Voice to Text   | Voice messages are fetched from Telegram and transcribed into text using OpenAI's Whisper API. |
| Transcribe Voice to Text | OpenAI (audio translate)         | Transcribes voice audio to text                   | Fetch Voice Message          | Basic LLM Chain            | Voice messages are fetched from Telegram and transcribed into text using OpenAI's Whisper API. |
| Prepare for LLM        | Set                              | Prepares text input for AI processing             | Voice or Text? (Text branch) | Basic LLM Chain            |                                                                                              |
| OpenAI Chat Model      | Langchain OpenAI Chat Model      | Provides GPT model for task analysis               | (Connected internally)       | Basic LLM Chain (language model) | The LLM (OpenAI Chat Model) analyzes the text and breaks it down into tasks and sub-tasks, formatted for Todoist. |
| Basic LLM Chain        | Langchain LLM Chain              | Sends prompt and text to AI, receives sub-task JSON | Prepare for LLM, Transcribe Voice to Text | Extract Tasks, Create Todoist Tasks | The LLM (OpenAI Chat Model) analyzes the text and breaks it down into tasks and sub-tasks, formatted for Todoist. |
| Extract Tasks          | Langchain Structured Output Parser | Parses AI response into structured JSON sub-tasks | Basic LLM Chain             | Create Todoist Tasks       | The LLM (OpenAI Chat Model) analyzes the text and breaks it down into tasks and sub-tasks, formatted for Todoist. |
| Create Todoist Tasks   | Todoist                          | Creates tasks in Todoist project                   | Extract Tasks               | Send Confirmation          |                                                                                              |
| Send Confirmation      | Telegram                         | Sends confirmation message back to Telegram user | Create Todoist Tasks        |                            |                                                                                              |
| Sticky Note2           | Sticky Note                     | Documentation note on input reception              |                             |                            | This workflow listens for incoming voice or text messages from Telegram users.               |
| Sticky Note3           | Sticky Note                     | Documentation note on voice message processing     |                             |                            | Voice messages are fetched from Telegram and transcribed into text using OpenAI's Whisper API. |
| Sticky Note4           | Sticky Note                     | Documentation note on AI task analysis             |                             |                            | The LLM (OpenAI Chat Model) analyzes the text and breaks it down into tasks and sub-tasks, formatted for Todoist. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure with your Telegram bot credentials  
   - Set to listen for "message" updates only  

2. **Add Switch Node ("Voice or Text?")**  
   - Type: Switch  
   - Add three outputs: "Audio", "Text", "Error"  
   - Audio condition: Check if `message.voice.file_id` exists  
   - Text condition: Check if `message.text` exists  
   - Error condition: Check if `error` field exists  

3. **Add Telegram Node ("Fetch Voice Message")**  
   - Type: Telegram  
   - Connect from "Audio" output of Switch  
   - Configure to fetch file using `message.voice.file_id`  

4. **Add OpenAI Node ("Transcribe Voice to Text")**  
   - Type: OpenAI (Langchain OpenAI node or standard OpenAI node)  
   - Set resource to "audio" and operation to "translate"  
   - Connect input from "Fetch Voice Message" node's binary data  
   - Configure with OpenAI API credentials  

5. **Add Set Node ("Prepare for LLM")**  
   - Type: Set  
   - Connect from "Text" output of Switch and from "Transcribe Voice to Text" node  
   - Assign field `text` with the value of the message text or transcribed text (`{{$json.message.text}}` or transcription output)  

6. **Add Langchain OpenAI Chat Model Node ("OpenAI Chat Model")**  
   - Type: Langchain OpenAI Chat Model  
   - Configure model to "gpt-4o-mini" (or preferred GPT model)  
   - Connect internally as language model for the next node  
   - Use OpenAI API credentials  

7. **Add Langchain LLM Chain Node ("Basic LLM Chain")**  
   - Type: Langchain LLM Chain  
   - Connect input from "Prepare for LLM" node  
   - Set text input to `={{ $json.text }}`  
   - Paste the detailed system prompt instructing AI to break down tasks into JSON with `content` and `priority` fields (as described in the prompt)  
   - Connect language model input to "OpenAI Chat Model" node  

8. **Add Langchain Structured Output Parser Node ("Extract Tasks")**  
   - Type: Langchain Structured Output Parser  
   - Connect input from "Basic LLM Chain" output  
   - Provide example JSON schema for validation (e.g., `{"content": "Send out invitations.", "priority": 3}`)  

9. **Add Todoist Node ("Create Todoist Tasks")**  
   - Type: Todoist  
   - Connect input from "Extract Tasks" node  
   - Set task content to `={{ $json.output.content }}`  
   - Set priority to `={{ $json.output.priority }}`  
   - Set project ID to your Todoist project (e.g., `"2349786654"`)  
   - Configure with Todoist API credentials  

10. **Add Telegram Node ("Send Confirmation")**  
    - Type: Telegram  
    - Connect input from "Create Todoist Tasks" node  
    - Set message text to: `Task : {{ $json.content }} Task Link :{{ $json.url }}`  
    - Set chat ID to `={{ $('Receive Telegram Messages').item.json.message.chat.id }}`  
    - Configure with Telegram API credentials  

11. **Add Sticky Notes (Optional for Documentation)**  
    - Add notes describing each block for clarity  

12. **Connect Nodes According to the Workflow Diagram**  
    - Receive Telegram Messages → Voice or Text?  
    - Voice or Text? (Audio) → Fetch Voice Message → Transcribe Voice to Text → Basic LLM Chain  
    - Voice or Text? (Text) → Prepare for LLM → Basic LLM Chain  
    - Basic LLM Chain → Extract Tasks → Create Todoist Tasks → Send Confirmation  

13. **Test the Workflow**  
    - Send voice and text messages to your Telegram bot  
    - Verify tasks are created in Todoist and confirmation messages are received  

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow combines Telegram, OpenAI, and Todoist to automate task management with AI-powered task breakdown. | Workflow description and use case summary                                                       |
| OpenAI Whisper API is used for voice transcription, enabling hands-free task creation.                      | https://openai.com/blog/whisper                                                                 |
| Todoist API integration allows direct task creation with priority settings.                                | https://developer.todoist.com/rest/v2/                                                           |
| Customize the LLM prompt to better fit your task decomposition needs.                                      | Prompt is embedded in the Basic LLM Chain node configuration                                    |
| Ensure API keys and credentials for Telegram, OpenAI, and Todoist are correctly configured before running. | n8n credential setup                                                                              |
| Explore Todoist features like projects, labels, and filters to enhance task organization.                   | https://todoist.com/features                                                                     |

---

This document provides a comprehensive understanding of the workflow, enabling reproduction, modification, and troubleshooting by both advanced users and AI agents.