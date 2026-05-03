🧑‍🎓 AI Powered Language Teacher with Telegram, Google Sheet and GPT-4o

https://n8nworkflows.xyz/workflows/------ai-powered-language-teacher-with-telegram--google-sheet-and-gpt-4o-3295


# 🧑‍🎓 AI Powered Language Teacher with Telegram, Google Sheet and GPT-4o

### 1. Workflow Overview

This workflow is an **AI-powered language tutor** designed to help users learn vocabulary in a target language (default example: Mandarin Chinese) through interactive multiple-choice questions (MCQs) delivered via a **Telegram Bot**. It leverages **Google Sheets** as the vocabulary source, **OpenAI GPT-4o** for question generation and answer evaluation, and maintains conversational context per user.

The workflow is logically divided into three main blocks:

- **1.1 Input Reception and Vocabulary Retrieval**: Triggered by Telegram messages, this block fetches the vocabulary list from Google Sheets and prepares it for AI processing.
- **1.2 AI Processing and Conversation Management**: Uses OpenAI GPT-4o via an AI Agent node to generate MCQs, evaluate user answers, and maintain conversation memory.
- **1.3 User Feedback Delivery**: Sends AI-generated questions and feedback back to the user via Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Vocabulary Retrieval

- **Overview:**  
  This block triggers the workflow upon receiving a Telegram message, retrieves the vocabulary list from a Google Sheet, and aggregates the vocabulary into two distinct lists (source and target language).

- **Nodes Involved:**  
  - Telegram Trigger  
  - Retrive Vocabulary (Google Sheets)  
  - Aggregate Vocabulary Lists

- **Node Details:**

  - **Telegram Trigger**  
    - *Type:* Trigger node for Telegram messages  
    - *Configuration:* Listens for new messages from users via a Telegram Bot webhook.  
    - *Key Variables:* Captures `message.text` and `message.chat.id` for user input and session identification.  
    - *Connections:* Output to "Retrive Vocabulary" node.  
    - *Potential Failures:* Telegram API connectivity issues, webhook misconfiguration, invalid bot token.  
    - *Notes:* Requires Telegram Bot credentials configured in n8n.

  - **Retrive Vocabulary**  
    - *Type:* Google Sheets node  
    - *Configuration:* Reads the vocabulary list from a specified Google Sheet document and sheet tab.  
    - *Key Variables:* Reads columns `initialText` (native language words) and `translatedText` (target language words).  
    - *Connections:* Output to "Aggregate Vocabulary Lists".  
    - *Potential Failures:* Google API authentication errors, incorrect document ID or sheet name, empty or malformed sheet data.  
    - *Notes:* Requires Google Sheets OAuth2 credentials with read access.

  - **Aggregate Vocabulary Lists**  
    - *Type:* Aggregate node  
    - *Configuration:* Aggregates the two vocabulary columns into separate arrays named `initialLanguage` and `targetLanguage`.  
    - *Key Variables:* Aggregates `initialText` → `initialLanguage`, `translatedText` → `targetLanguage`.  
    - *Connections:* Output to "AI Agent".  
    - *Potential Failures:* Empty input data, aggregation misconfiguration.

---

#### 2.2 AI Processing and Conversation Management

- **Overview:**  
  This block generates interactive MCQs based on the vocabulary list, evaluates user responses, and maintains session memory to provide a continuous learning experience.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Simple Memory

- **Node Details:**

  - **AI Agent**  
    - *Type:* LangChain AI Agent node  
    - *Configuration:*  
      - Receives user input text from Telegram and vocabulary lists from aggregation.  
      - Uses a detailed system prompt to instruct GPT-4o to generate MCQs, evaluate answers, and provide feedback.  
      - The prompt includes instructions for question formatting, evaluation logic, and user interaction rules.  
      - Session key is set per Telegram chat ID to maintain user-specific conversation context.  
    - *Key Expressions:*  
      - Input text: `={{ $('Telegram Trigger').item.json.message.text }}`  
      - Vocabulary lists accessed via `$json.targetLanguage` inside the prompt.  
    - *Connections:*  
      - Input: Receives memory from "Simple Memory" and language model from "OpenAI Chat Model".  
      - Output: Sends response text to "Answer to the User".  
    - *Potential Failures:*  
      - OpenAI API rate limits or authentication errors.  
      - Prompt formatting errors or unexpected user inputs.  
      - Memory session key conflicts or data loss.  
    - *Notes:*  
      - Requires OpenAI API credentials configured in the "OpenAI Chat Model" node.

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Model node  
    - *Configuration:* Uses GPT-4o-mini model for chat completions.  
    - *Connections:* Provides language model interface to "AI Agent".  
    - *Potential Failures:* API key invalid, model unavailability, network issues.

  - **Simple Memory**  
    - *Type:* LangChain Memory Buffer Window node  
    - *Configuration:* Maintains conversation history per Telegram chat ID (`sessionKey` set to chat ID).  
    - *Connections:* Feeds memory context into "AI Agent".  
    - *Potential Failures:* Memory overflow or session key misconfiguration.

---

#### 2.3 User Feedback Delivery

- **Overview:**  
  This block sends the AI-generated questions and feedback messages back to the user on Telegram.

- **Nodes Involved:**  
  - Answer to the User (Telegram node)

- **Node Details:**

  - **Answer to the User**  
    - *Type:* Telegram node (send message)  
    - *Configuration:*  
      - Sends text output from the AI Agent back to the Telegram chat identified by `chat.id`.  
      - Does not append attribution to messages.  
    - *Key Expressions:*  
      - Message text: `={{ $json.output }}` (output from AI Agent)  
      - Chat ID: `={{ $('Telegram Trigger').item.json.message.chat.id }}`  
    - *Potential Failures:* Telegram API errors, invalid chat ID, message formatting issues.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                          | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                          |
|-------------------------|--------------------------------------|----------------------------------------|------------------------|------------------------|----------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger        | n8n-nodes-base.telegramTrigger        | Workflow trigger on user Telegram msg  | —                      | Retrive Vocabulary      | ### 1. Workflow Trigger with Telegram Message<br>Setup Telegram Bot credentials and Google Sheets API credentials.   |
| Retrive Vocabulary      | n8n-nodes-base.googleSheets           | Fetch vocabulary list from Google Sheet| Telegram Trigger       | Aggregate Vocabulary Lists | See above                                                                                                            |
| Aggregate Vocabulary Lists | n8n-nodes-base.aggregate             | Aggregate vocabulary columns into lists| Retrive Vocabulary     | AI Agent               | See above                                                                                                            |
| AI Agent                | @n8n/n8n-nodes-langchain.agent        | Generate MCQs, evaluate answers, manage conversation | Aggregate Vocabulary Lists, Simple Memory, OpenAI Chat Model | Answer to the User | ### 2. Conversational AI Agent<br>Set up OpenAI GPT-4o-mini and adapt system prompt for target language.             |
| OpenAI Chat Model       | @n8n/n8n-nodes-langchain.lmChatOpenAi | Provides GPT-4o language model         | —                      | AI Agent               | See above                                                                                                            |
| Simple Memory           | @n8n/n8n-nodes-langchain.memoryBufferWindow | Maintains conversation memory per user | —                      | AI Agent               | See above                                                                                                            |
| Answer to the User      | n8n-nodes-base.telegram               | Sends AI-generated messages to Telegram chat | AI Agent               | —                      | See above                                                                                                            |
| Sticky Note1            | n8n-nodes-base.stickyNote             | Instructional note for block 1          | —                      | —                      | ### 1. Workflow Trigger with Telegram Message<br>Setup instructions for Telegram and Google Sheets nodes.            |
| Sticky Note             | n8n-nodes-base.stickyNote             | Instructional note for block 2          | —                      | —                      | ### 2. Conversational AI Agent<br>Setup instructions for AI Agent and OpenAI Chat Model nodes.                        |
| Sticky Note3            | n8n-nodes-base.stickyNote             | Tutorial and detailed guide link         | —                      | —                      | ### 3. Do you need more details?<br>[🎥 Watch My Tutorial](https://youtu.be/MQV8wDSug7M)                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: `Telegram Trigger`  
   - Configure with your Telegram Bot credentials (token).  
   - Set to listen for `message` updates.  
   - This node triggers the workflow on any new user message.

2. **Create Google Sheets Node (Retrive Vocabulary)**  
   - Type: `Google Sheets`  
   - Configure OAuth2 credentials for Google Sheets API.  
   - Select the Google Sheet document containing your vocabulary list.  
   - Select the specific sheet/tab with the vocabulary.  
   - Ensure the sheet has columns named `initialText` (native language) and `translatedText` (target language).  
   - Connect input from Telegram Trigger node.

3. **Create Aggregate Node (Aggregate Vocabulary Lists)**  
   - Type: `Aggregate`  
   - Configure to aggregate two fields:  
     - `initialText` → output field `initialLanguage`  
     - `translatedText` → output field `targetLanguage`  
   - Connect input from Google Sheets node.

4. **Create OpenAI Chat Model Node**  
   - Type: `LangChain OpenAI Chat Model`  
   - Configure with OpenAI API credentials.  
   - Select model `gpt-4o-mini`.  
   - No additional options required.  
   - This node provides the language model for the AI Agent.

5. **Create Simple Memory Node**  
   - Type: `LangChain Memory Buffer Window`  
   - Set `sessionKey` to `={{ $('Telegram Trigger').item.json.message.chat.id }}` to maintain per-user session.  
   - This node manages conversation history.

6. **Create AI Agent Node**  
   - Type: `LangChain Agent`  
   - Configure input text as `={{ $('Telegram Trigger').item.json.message.text }}`.  
   - Set system prompt to instruct GPT-4o to:  
     - Generate MCQs from vocabulary lists.  
     - Evaluate user answers.  
     - Provide feedback and continue questioning.  
     - Use the vocabulary lists aggregated (`initialLanguage` and `targetLanguage`) inside the prompt.  
   - Connect AI Agent inputs:  
     - `ai_languageModel` input from OpenAI Chat Model node.  
     - `ai_memory` input from Simple Memory node.  
     - Main input from Aggregate Vocabulary Lists node.  
   - Connect output to Telegram node for sending answers.

7. **Create Telegram Node (Answer to the User)**  
   - Type: `Telegram` (send message)  
   - Configure to send message text from AI Agent output: `={{ $json.output }}`.  
   - Set `chatId` to `={{ $('Telegram Trigger').item.json.message.chat.id }}`.  
   - Connect input from AI Agent node.

8. **Add Sticky Notes for Documentation (Optional)**  
   - Add sticky notes near each block to provide setup instructions and links as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow supports **any language** by modifying the AI Agent system prompt to your target language. | Workflow flexibility                                                                                 |
| Tutorial video for detailed setup and usage: [🎥 Watch My Tutorial](https://youtu.be/MQV8wDSug7M)    | Visual guide for users                                                                            |
| Vocabulary list generation template available: [Generate Anki Flash Cards for Language Learning](https://n8n.io/workflows/3195-generate-anki-flash-cards-for-language-learning-with-google-translate-and-gpt-4o/) | To create Google Sheets vocabulary lists                                                        |
| Workflow created with n8n version 1.82.1 on March 23, 2025                                          | Version compatibility                                                                            |
| LinkedIn profile of creator for networking: [Samir Saci](https://www.linkedin.com/in/samir-saci)    | Professional contact                                                                             |

---

This documentation provides a complete, structured understanding of the AI-powered language tutor workflow, enabling reproduction, modification, and troubleshooting by advanced users and AI agents alike.