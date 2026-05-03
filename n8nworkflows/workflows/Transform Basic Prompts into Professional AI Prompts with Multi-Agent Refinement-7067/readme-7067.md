Transform Basic Prompts into Professional AI Prompts with Multi-Agent Refinement

https://n8nworkflows.xyz/workflows/transform-basic-prompts-into-professional-ai-prompts-with-multi-agent-refinement-7067


# Transform Basic Prompts into Professional AI Prompts with Multi-Agent Refinement

### 1. Workflow Overview

This workflow automates the transformation of basic user prompts into professionally enhanced AI prompts using multi-agent refinement. It is targeted at users needing to improve prompt clarity, structure, and detail for AI-driven content creation, marketing, technical documentation, or educational purposes.

The workflow is divided into four logical blocks:

- **1.1 Input Reception and Preparation:**  
  Triggered by new entries in a Google Sheets tab containing original prompts, this block reads, batches, and prepares prompt data for processing.

- **1.2 AI Prompt Enhancement:**  
  Uses multi-agent AI models (Google Gemini, Groq Qwen) to analyze and enrich prompts with more detail, structure, and contextual clarity. Incorporates chat history and system messages to guide AI behavior.

- **1.3 Parsing and Validation:**  
  Parses AI output robustly, extracting JSON data with fallback parsing strategies. Validates completeness of enhanced prompts and manages retries with capped execution attempts.

- **1.4 Output Storage and Notification:**  
  Saves the refined prompts back into a Google Sheets tab with metadata and sends success or error notifications via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preparation

- **Overview:**  
  Monitors a Google Sheets tab for new or updated original prompts and prepares them for processing one at a time.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Loop Over Items  
  - PrepareForFirstGen - OPM

- **Node Details:**  

  - **Google Sheets Trigger**  
    - Type: Google Sheets Trigger  
    - Role: Watches a specific sheet/tab for changes in columns "Original Prompt ID" and "Original Prompt" every minute.  
    - Configuration: Polls every minute on the designated Google Sheet and tab (gid=0). Uses OAuth2 credentials.  
    - Inputs: None (trigger)  
    - Outputs: New or changed prompt rows as JSON  
    - Edge Cases: OAuth token expiration, sheet access permissions, no new data found.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Splits batch of prompts into single items for sequential processing to avoid overload.  
    - Configuration: Default batch size (1) implied, no options set.  
    - Inputs: Data from Google Sheets Trigger  
    - Outputs: Single items forwarded downstream  
    - Edge Cases: Large batch sizes could cause delays or memory issues.

  - **PrepareForFirstGen - OPM**  
    - Type: Set  
    - Role: Prepares and enriches input data with flags and session info for AI processing.  
    - Configuration: Copies "Original Prompt ID", passes existing chatInput if any, sets isProcessed false, sessionId 0, and adds systemMessage from input JSON.  
    - Inputs: Single prompt item from Loop Over Items  
    - Outputs: Structured JSON ready for AI agent  
    - Edge Cases: Missing input fields, null systemMessage.

---

#### 2.2 AI Prompt Enhancement

- **Overview:**  
  Runs AI agents (Google Gemini and Groq Qwen) with prompt context, system messages, and chat history to generate enhanced prompt content.

- **Nodes Involved:**  
  - ExecutionController  
  - ChatHistory - PM  
  - Gemini - PM  
  - Qwen3 - PM  
  - PromptModifer (Agent)

- **Node Details:**

  - **ExecutionController**  
    - Type: Code  
    - Role: Controls execution flow to prevent duplicate or excessive AI agent runs per prompt, manages retry counts, and builds dynamic system messages guiding AI behavior.  
    - Configuration:  
      - Tracks prompt execution counts with global static data to limit retries to max 3.  
      - Builds different system messages depending on retry attempt number.  
      - Throws error if retry limit exceeded.  
    - Inputs: Prepared prompt JSON from PrepareForFirstGen - OPM  
    - Outputs: JSON with sessionId, chatInput, systemMessage, isProcessed, and counts for next nodes  
    - Edge Cases: Errors in static data storage, infinite loops prevented by careful key management.

  - **ChatHistory - PM**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains a window of the last 4 conversational exchanges to provide context for the AI models.  
    - Configuration: contextWindowLength=4  
    - Inputs: ExecutionController output  
    - Outputs: Context-enriched messages  
    - Edge Cases: Empty conversation history on first run, memory buffer overflow.

  - **Gemini - PM**  
    - Type: LangChain Google Gemini Chat Model  
    - Role: Runs Google Gemini Pro 2.5 AI model to generate enhanced prompt outputs based on system message and chat inputs.  
    - Configuration: Uses Google Palm API credential, default options.  
    - Inputs: Context-enriched data from ChatHistory - PM  
    - Outputs: AI chat response with enhanced prompt JSON inside  
    - Edge Cases: API authentication failures, rate limiting, model downtime.

  - **Qwen3 - PM**  
    - Type: LangChain Groq Qwen Chat Model  
    - Role: Alternative AI model to Gemini for prompt refinement, providing redundancy or comparison.  
    - Configuration: Model "qwen/qwen3-32b" with Groq API credential.  
    - Inputs: Context-enriched data from ChatHistory - PM  
    - Outputs: AI chat response  
    - Edge Cases: Same as Gemini - PM.

  - **PromptModifer**  
    - Type: LangChain Agent  
    - Role: Orchestrates AI chat generation using system messages and input data; supports fallback if AI fails.  
    - Configuration: Receives dynamic system messages from ExecutionController, runs AI agent with fallback enabled.  
    - Inputs: From Gemini - PM or Qwen3 - PM AI model outputs  
    - Outputs: AI-generated prompt enhancement JSON text  
    - Edge Cases: Agent fallback triggering if AI returns errors or incomplete data.

---

#### 2.3 Parsing and Validation

- **Overview:**  
  Parses and validates AI-generated prompt JSON output, ensuring all required fields are present, and manages retries or errors accordingly.

- **Nodes Involved:**  
  - Parse AI Chat  
  - Switch

- **Node Details:**

  - **Parse AI Chat**  
    - Type: Code  
    - Role: Parses AI output text to extract structured JSON data with multiple fallback parsing methods (JSON block, JSON object pattern, full JSON parse).  
    - Configuration:  
      - Initializes parsed data with default error values.  
      - Tries three parsing methods for robustness.  
      - Extracts fields like modified_prompt, prompt_type, topic, prompt_title, topic_categories, improvement_notes.  
      - Sets parsing success flag accordingly.  
    - Inputs: AI agent output from PromptModifer  
    - Outputs: Parsed JSON with prompt fields and metadata  
    - Edge Cases: Malformed AI output, missing JSON block, partial data, parsing errors.

  - **Switch**  
    - Type: Switch  
    - Role: Routes parsed data based on field completeness and retry counts into three paths: all data valid, missing data needing retry, or exceeding retry limits.  
    - Configuration:  
      - Checks for missing required fields and operation counts.  
      - Outputs:  
        - allDatavalid → Finalize ModifiedPrompt DataModel  
        - missingData → ExecutionController (retry path)  
        - exceedLimits → Send Exceed Limits Error  
    - Inputs: Parsed JSON from Parse AI Chat  
    - Outputs: Conditional routing to next nodes  
    - Edge Cases: Logic errors in condition expressions, unexpected field states.

---

#### 2.4 Output Storage and Notification

- **Overview:**  
  Finalizes prompt data model, writes enhanced prompt data back to Google Sheets, and notifies users via Telegram of success or failure.

- **Nodes Involved:**  
  - Finalize ModifiedPrompt DataModel  
  - Create/Upate Modified Prompt  
  - Send Success Message To User  
  - Send Exceed Limits Error

- **Node Details:**

  - **Finalize ModifiedPrompt DataModel**  
    - Type: Set  
    - Role: Structures and flags the final enhanced prompt data, converting fields into the expected schema for storage.  
    - Configuration: Sets isProcessed=true, updates timestamps, copies all required fields from parsed JSON.  
    - Inputs: Validated parsed data from Switch (allDatavalid path)  
    - Outputs: Ready-to-store prompt JSON  
    - Edge Cases: Missing or inconsistent field data.

  - **Create/Upate Modified Prompt**  
    - Type: Google Sheets  
    - Role: Appends or updates the "Modified Prompt Sheet" tab in the Google Spreadsheet with enhanced prompt data.  
    - Configuration: Matches rows by Original Prompt ID, updates columns including Topic, Prompt Title, Modified Prompt, Improvement Notes, timestamps, and processing flags. Uses OAuth2 credentials.  
    - Inputs: Finalized prompt data from Finalize ModifiedPrompt DataModel  
    - Outputs: Confirmation of sheet update  
    - Edge Cases: API quota limits, sheet permission issues, concurrent write conflicts.

  - **Send Success Message To User**  
    - Type: Telegram  
    - Role: Sends a Telegram message notifying that prompt refinement completed successfully, including summary info and a preview snippet.  
    - Configuration: Uses Telegram API credentials, sends to configured chat ID, message includes prompt title, topic, categories, partial prompt preview, model used, and completion time.  
    - Inputs: Output from Loop Over Items (after processing)  
    - Outputs: Telegram message sent confirmation  
    - Edge Cases: Telegram API rate limits, invalid chat ID.

  - **Send Exceed Limits Error**  
    - Type: Telegram  
    - Role: Sends an error notification via Telegram if prompt processing failed due to retry limit exceeded.  
    - Configuration: Message details include prompt ID, original prompt snippet, issue description, and troubleshooting suggestions.  
    - Inputs: Switch node path exceedLimits  
    - Outputs: Telegram message sent confirmation  
    - Edge Cases: Same as Send Success Message To User.

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                              | Input Node(s)                   | Output Node(s)                   | Sticky Note                                                                                               |
|----------------------------|---------------------------------|----------------------------------------------|--------------------------------|---------------------------------|-----------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger       | Google Sheets Trigger            | Watches for new/changed original prompts    | None                           | Loop Over Items                 |                                                                                                           |
| Loop Over Items             | SplitInBatches                  | Processes prompts one by one                  | Google Sheets Trigger          | Send Success Message To User, ExecutionController |                                                                                                           |
| PrepareForFirstGen - OPM    | Set                             | Prepares data for AI processing               | ExecutionController            | PromptModifer                  |                                                                                                           |
| ExecutionController         | Code                            | Controls execution flow and retry limits     | Loop Over Items                | PrepareForFirstGen - OPM       |                                                                                                           |
| ChatHistory - PM            | LangChain Memory Buffer Window  | Maintains recent chat context                  | ExecutionController            | Gemini - PM, Qwen3 - PM        |                                                                                                           |
| Gemini - PM                | LangChain AI Chat Model          | Runs Google Gemini AI prompt enhancement      | ChatHistory - PM               | PromptModifer                  |                                                                                                           |
| Qwen3 - PM                 | LangChain AI Chat Model          | Runs Groq Qwen AI prompt enhancement          | ChatHistory - PM               | PromptModifer                  |                                                                                                           |
| PromptModifer              | LangChain Agent                  | Orchestrates AI prompt refinement             | Gemini - PM, Qwen3 - PM        | Parse AI Chat                  |                                                                                                           |
| Parse AI Chat              | Code                            | Parses AI output, extracts structured JSON    | PromptModifer                 | Switch                        |                                                                                                           |
| Switch                    | Switch                          | Routes parsed data based on completeness      | Parse AI Chat                 | Finalize ModifiedPrompt DataModel, ExecutionController, Send Exceed Limits Error |                                                                                                           |
| Finalize ModifiedPrompt DataModel | Set                       | Prepares final prompt data for storage        | Switch (allDatavalid)          | Create/Upate Modified Prompt    |                                                                                                           |
| Create/Upate Modified Prompt | Google Sheets                  | Updates Google Sheets with enhanced prompts   | Finalize ModifiedPrompt DataModel | None                        |                                                                                                           |
| Send Success Message To User | Telegram                      | Sends success notification to user via Telegram | Loop Over Items               | None                         |                                                                                                           |
| Send Exceed Limits Error    | Telegram                      | Sends error notification when retry limit exceeded | Switch (exceedLimits)          | None                         |                                                                                                           |
| Sticky Note                | Sticky Note                    | Workflow introduction and demo                | None                           | None                          | "🤖 AI-Powered Prompt Enhancement Assistant..." (full content in node)                                    |
| Sticky Note1               | Sticky Note                    | Step-by-step setup guide                       | None                           | None                          | "🛠️ STEP-BY-STEP SETUP GUIDE" (full content in node)                                                     |
| Sticky Note2               | Sticky Note                    | Workflow flow explanation                       | None                           | None                          | "🔄 WORKFLOW FLOW EXPLAINED" (full content in node)                                                      |
| Sticky Note3               | Sticky Note                    | Large empty sticky note (visual layout aid)   | None                           | None                          |                                                                                                           |
| Sticky Note4               | Sticky Note                    | Customization options and troubleshooting tips | None                           | None                          | "🔧 CUSTOMIZATION OPTIONS" and "🩺 TROUBLESHOOTING" (full content in node)                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets:**

   - Create a Google Spreadsheet with two sheets/tabs:
     - **Main Sheet:** Columns: Original Prompt ID, Model, Original Prompt, Created Time
     - **Modified Prompt Sheet:** Columns: Modified Prompt ID, Original Prompt ID, Topic, Topic Categories, Modified Prompt, Prompt Title, Prompt Type, Model Used, Improvement Notes, Updated Time, Created Time, isProcessed

2. **Set Up Credentials in n8n:**

   - Add credentials for:
     - Google Sheets OAuth2 (with access to the created spreadsheet)
     - Google Gemini (Google PaLM API)
     - Groq API (Qwen model)
     - Telegram API (for notifications)

3. **Create Nodes:**

   - **Google Sheets Trigger:**  
     - Type: Google Sheets Trigger  
     - Configure to watch the Main Sheet tab (gid=0) of your spreadsheet  
     - Set to poll every minute  
     - Watch columns: Original Prompt ID, Original Prompt

   - **Loop Over Items:**  
     - Type: SplitInBatches  
     - No special options needed

   - **ExecutionController (Code):**  
     - Type: Code  
     - Add JavaScript code to track prompt executions, prevent duplicates, limit retries to 3, and generate system messages for AI agents based on retry count  
     - Inputs: Output from Loop Over Items

   - **PrepareForFirstGen - OPM (Set):**  
     - Copy "Original Prompt ID"  
     - Pass "chatInput" from input or blank  
     - Set "isProcessed" boolean to false  
     - Initialize "sessionId" to 0  
     - Pass "systemMessage" from input

   - **ChatHistory - PM (LangChain Memory Buffer):**  
     - Configure contextWindowLength=4

   - **Gemini - PM (LangChain AI Chat):**  
     - Select Google Gemini (PaLM) model  
     - Use Google PaLM API credentials

   - **Qwen3 - PM (LangChain AI Chat):**  
     - Select Groq Qwen3-32b model  
     - Use Groq API credentials

   - **PromptModifer (LangChain Agent):**  
     - Set to use systemMessage from ExecutionController  
     - Enable fallback  
     - Inputs from Gemini - PM and Qwen3 - PM outputs

   - **Parse AI Chat (Code):**  
     - JavaScript code that tries to parse AI output with three fallback methods to extract JSON  
     - Extract fields: modified_prompt, prompt_type, prompt_title, topic, topic_categories, improvement_notes  
     - Set flags for parsing success and operation counts

   - **Switch:**  
     - Define conditions to check if all required fields are present and operation counts <= 2 (allDatavalid)  
     - Route to ExecutionController again if data missing and operation counts <= 2 (missingData)  
     - Route to Send Exceed Limits Error if operation counts >= 3 (exceedLimits)

   - **Finalize ModifiedPrompt DataModel (Set):**  
     - Set isProcessed = true  
     - Copy all prompt fields from parsed JSON  
     - Update last updated timestamp

   - **Create/Upate Modified Prompt (Google Sheets):**  
     - Operation: appendOrUpdate  
     - Match on Original Prompt ID  
     - Write to Modified Prompt Sheet tab (gid for this tab)  
     - Map all prompt fields accordingly

   - **Send Success Message To User (Telegram):**  
     - Configure Telegram API credentials  
     - Set chat ID for user notification  
     - Compose message with prompt title, topic, categories, partial modified prompt preview, model used, and time

   - **Send Exceed Limits Error (Telegram):**  
     - Configure Telegram API credentials  
     - Set chat ID  
     - Compose error message explaining retry limit exceeded and troubleshooting steps

4. **Connect Nodes in Order:**

   - Google Sheets Trigger → Loop Over Items  
   - Loop Over Items → Send Success Message To User (side output)  
   - Loop Over Items → ExecutionController  
   - ExecutionController → PrepareForFirstGen - OPM  
   - PrepareForFirstGen - OPM → PromptModifer (AI Agent)  
   - ChatHistory - PM connects between ExecutionController and AI models (Gemini and Qwen3) feeding PromptModifer  
   - PromptModifer → Parse AI Chat  
   - Parse AI Chat → Switch  
   - Switch outputs:  
     - allDatavalid → Finalize ModifiedPrompt DataModel → Create/Upate Modified Prompt  
     - missingData → ExecutionController (retry)  
     - exceedLimits → Send Exceed Limits Error

5. **Set Up Polling and Testing:**

   - Ensure polling interval is set to 1 minute in Google Sheets Trigger  
   - Add test prompt to Main Sheet to verify full flow  
   - Confirm success notification and data written to Modified Prompt Sheet

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                 |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------|
| 🤖 AI-Powered Prompt Enhancement Assistant demo example: Input "Create a marketing strategy" outputs a detailed, structured marketing prompt with variables for reusability.                                                                                                                                                                                   | Sticky Note at workflow start                                                                                  |
| 🛠️ Setup guide instructs creating Google Sheets with exact columns, configuring credentials, updating document IDs, and testing with sample prompts.                                                                                                                                                                                                       | Sticky Note1                                                                                                   |
| 🔄 Workflow flow explanation clarifies stages: Input → AI Enhancement → Validation → Storage & Notification → Error Handling with execution limits and graceful fallbacks.                                                                                                                                                                                  | Sticky Note2                                                                                                   |
| 🔧 Customization options available for AI behavior, retry limits, batch sizes, and Google Sheets columns. Tips for troubleshooting common issues including infinite loops, API errors, and missing fields are documented.                                                                                                                                     | Sticky Note4                                                                                                   |
| Workflow respects API rate limits by splitting items and limiting AI agent retries to max 3. Robust parsing protects against malformed AI outputs.                                                                                                                                                                                                         | Workflow design best practice                                                                                  |
| Telegram notifications provide real-time feedback on workflow status, aiding monitoring and quick issue detection.                                                                                                                                                                                                                                          | Notification nodes                                                                                             |
| System messages for AI are dynamically generated based on retry counts to guide the AI agents towards completing missing fields and improving prompt quality.                                                                                                                                                                                              | ExecutionController code node                                                                                  |
| Workflow is designed for professional prompt engineering workflows including marketing, content creation, educational prompts, and technical documentation enhancement scenarios.                                                                                                                                                                         | Workflow description                                                                                           |

---

**Disclaimer:**  
The provided text is derived exclusively from an n8n automated workflow. All processing complies with current content policies and handles only legal, public data without any illegal or offensive elements.