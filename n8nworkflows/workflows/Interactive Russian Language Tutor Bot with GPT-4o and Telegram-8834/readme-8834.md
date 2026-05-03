Interactive Russian Language Tutor Bot with GPT-4o and Telegram

https://n8nworkflows.xyz/workflows/interactive-russian-language-tutor-bot-with-gpt-4o-and-telegram-8834


# Interactive Russian Language Tutor Bot with GPT-4o and Telegram

---

### 1. Workflow Overview

This workflow implements an **Interactive Russian Language Tutor Bot** using n8n, integrating Telegram with GPT-4o AI for personalized language learning. It serves users who want help with Russian vocabulary, grammar, or quizzes via chat on Telegram. The user interacts by sending messages tagged with hashtags (#vocabulary, #grammar, #quiz) or plain text for general assistance.

The workflow's logic is structured into these blocks:

- **1.1 Input Reception and Typing Indicator:** Receives incoming Telegram messages and immediately shows a "typing..." status to enhance user experience.
- **1.2 Input Routing:** Examines the user's message text for hashtags to determine the learning mode (Vocabulary, Grammar, Quiz, or Other).
- **1.3 Prompt Preparation:** Builds tailored prompts for GPT-4o according to the learning mode or general interaction.
- **1.4 AI Processing:** Sends the prepared prompt plus user message to the GPT-4o model to generate a personalized teaching response.
- **1.5 Output Delivery:** Sends the AI-generated response back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Typing Indicator

- **Overview:**  
  Captures incoming messages from Telegram and immediately sends a "typing..." indicator in the chat to simulate human-like responsiveness.

- **Nodes Involved:**  
  - Start: Receive Message on Telegram  
  - Show Typing Indicator

- **Node Details:**

  - **Start: Receive Message on Telegram**  
    - *Type:* Telegram Trigger  
    - *Role:* Entry point listening to new messages from Telegram users.  
    - *Configuration:* Listens for "message" update type only. Requires Telegram bot token credential configured in n8n.  
    - *Expressions:* Uses incoming message text and chat ID for downstream logic.  
    - *Connections:* Output connected to Show Typing Indicator.  
    - *Edge Cases:* Telegram API downtime, invalid bot token, or malformed messages may cause missed triggers.

  - **Show Typing Indicator**  
    - *Type:* Telegram node  
    - *Role:* Sends chat action "typing" to the user chat to show the bot is processing.  
    - *Configuration:* Uses chat ID from the incoming Telegram message.  
    - *Expressions:* `chatId = {{ $('Start: Receive Message on Telegram').item.json.message.chat.id }}`  
    - *Connections:* Output connected to Route by Input Type switch node.  
    - *Edge Cases:* Telegram API rate limits or invalid chat IDs may cause failure to show typing status.

#### 2.2 Input Routing

- **Overview:**  
  Routes the incoming message based on presence and type of hashtag tags (#vocabulary, #grammar, #quiz) to select the appropriate prompt-building node or defaults to a general prompt.

- **Nodes Involved:**  
  - Route by Input Type (Switch)  
  - Sticky Note1 (comment)

- **Node Details:**

  - **Route by Input Type**  
    - *Type:* Switch node  
    - *Role:* Parses message text to detect specific hashtags at message start and directs flow accordingly.  
    - *Configuration:* Four outputs: Vocabulary (starts with #vocabulary), Grammar (starts with #grammar), Quiz (starts with #quiz), and Any other (fallback for all other messages).  
    - *Expressions:* Uses expression evaluating `$('Start: Receive Message on Telegram').item.json.message.text` with string operations `startsWith`.  
    - *Connections:*  
      - Vocabulary → Vocabulary prompt  
      - Grammar → Grammar prompt  
      - Quiz → Quiz prompt  
      - Any other → Main prompt for all other messages  
    - *Edge Cases:* Misspelled hashtags or messages with hashtags not at start fall through to "Any other". Empty or non-text messages might cause expression errors.

  - **Sticky Note1**  
    - *Purpose:* Notes the switch node’s logic to check for hashtags that determine routing.

#### 2.3 Prompt Preparation

- **Overview:**  
  Prepares specific prompt texts for each learning mode to instruct the AI how to respond. Each prompt defines the AI role, output style, and expected content.

- **Nodes Involved:**  
  - Vocabulary prompt  
  - Grammar prompt  
  - Quiz prompt  
  - Main prompt for all other messages

- **Node Details:**

  - **Vocabulary prompt**  
    - *Type:* Set node  
    - *Role:* Sets a tailored prompt to teach vocabulary words with pronunciation, examples, and practice tasks.  
    - *Configuration:* Raw JSON with a detailed prompt text instructing GPT-4o to provide exactly the requested number of Russian words and use rich formatting suitable for Telegram. Ends with interactive tasks and encouragement.  
    - *Edge Cases:* If user input doesn’t include a number of words, AI may not parse quantity correctly.

  - **Grammar prompt**  
    - *Type:* Set node  
    - *Role:* Provides a prompt to explain grammar rules with examples and exercises, encouraging the student to practice.  
    - *Configuration:* Prompt text instructs AI to adjust explanation depth based on user request and include corrections.  
    - *Edge Cases:* Vague or ambiguous grammar topics might yield less precise AI responses.

  - **Quiz prompt**  
    - *Type:* Set node  
    - *Role:* Creates quiz questions and exercises formatted for Telegram, including answers and feedback.  
    - *Configuration:* Specifies quiz style (multiple-choice, fill-in-the-blank) and positive feedback in Russian.  
    - *Edge Cases:* User may request zero or extremely high number of questions, potentially causing AI overload or empty results.

  - **Main prompt for all other messages**  
    - *Type:* Set node  
    - *Role:* General fallback prompt that asks the student to choose a mode if none is specified. Delivers general supportive answers.  
    - *Configuration:* Instructs AI to adapt response style based on hashtag presence and always explain clearly with examples.  
    - *Edge Cases:* Plain text messages without clear intent may lead to generic or unhelpful AI responses.

#### 2.4 AI Processing

- **Overview:**  
  Sends the constructed prompt and user message to OpenAI GPT-4o model to generate a personalized answer.

- **Nodes Involved:**  
  - Generate personalised answer

- **Node Details:**

  - **Generate personalised answer**  
    - *Type:* n8n AIML API node (OpenAI GPT-4o integration)  
    - *Role:* Executes the OpenAI API call with the prompt and incoming message to get AI-generated content.  
    - *Configuration:* Uses model "openai/gpt-4o", prompt is concatenation of the prompt set node's output plus the original message text.  
    - *Expressions:*  
      - Prompt input: `{{$json.prompt}}` + user message text appended.  
    - *Connections:* Output connected to Send message to Telegram node.  
    - *Credentials:* Requires OpenAI API key configured in n8n.  
    - *Edge Cases:* API quota exceeded, network issues, malformed prompt, or model errors can cause failures or delays.

#### 2.5 Output Delivery

- **Overview:**  
  Sends the AI-generated response message back to the Telegram chat.

- **Nodes Involved:**  
  - Send message to Telegram

- **Node Details:**

  - **Send message to Telegram**  
    - *Type:* Telegram node  
    - *Role:* Delivers the text content generated by the AI back to the user chat in Telegram.  
    - *Configuration:*  
      - `text` parameter set from AI output content (JSON field `content`).  
      - `chatId` taken from incoming message chat ID.  
      - `appendAttribution` disabled to prevent default bot signature.  
    - *Edge Cases:* Invalid chat ID, Telegram API rate limit, or message length limits may cause delivery failure.

---

### 3. Summary Table

| Node Name                      | Node Type                       | Functional Role                        | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                       |
|--------------------------------|--------------------------------|-------------------------------------|--------------------------------|-------------------------------|------------------------------------------------------------------|
| Start: Receive Message on Telegram | Telegram Trigger               | Entry point, receives Telegram messages | —                              | Show Typing Indicator          |                                                                  |
| Show Typing Indicator           | Telegram node                  | Shows “typing…” status in chat      | Start: Receive Message on Telegram | Route by Input Type            | Checks for messages and emulates typing                          |
| Route by Input Type             | Switch                        | Routes message based on hashtag     | Show Typing Indicator           | Vocabulary prompt, Grammar prompt, Quiz prompt, Main prompt for all other messages | Checks message.text for #vocabulary, #grammar, #quiz, or falls back |
| Vocabulary prompt              | Set node                      | Builds vocabulary teaching prompt   | Route by Input Type (Vocabulary) | Generate personalised answer   |                                                                  |
| Grammar prompt                 | Set node                      | Builds grammar explanation prompt   | Route by Input Type (Grammar)  | Generate personalised answer   |                                                                  |
| Quiz prompt                   | Set node                      | Builds quiz creation prompt         | Route by Input Type (Quiz)     | Generate personalised answer   |                                                                  |
| Main prompt for all other messages | Set node                      | Builds default/general prompt       | Route by Input Type (Any other) | Generate personalised answer   |                                                                  |
| Generate personalised answer   | AIML API (OpenAI GPT-4o)       | Calls GPT-4o to generate response   | Vocabulary prompt, Grammar prompt, Quiz prompt, Main prompt for all other messages | Send message to Telegram       | Generates and sends answer to Telegram                           |
| Send message to Telegram       | Telegram node                  | Sends AI-generated message to user | Generate personalised answer   | —                             |                                                                  |
| Sticky Note                    | Sticky Note                   | Instructions and tips for setup     | —                              | —                             | 📖 Getting Started in 4 Steps                                    |
| Sticky Note1                   | Sticky Note                   | Notes hashtag routing logic         | —                              | —                             | Checks message.text for #vocabulary, #grammar, #quiz, or falls back |
| Sticky Note2                   | Sticky Note                   | Node overview summary                | —                              | —                             | 🔍 Node Overview: Telegram Trigger, Typing Indicator, Router, Prompts, Send message |
| Sticky Note3                   | Sticky Note                   | Notes prompt selection logic        | —                              | —                             | Chooses from given prompts                                       |
| Sticky Note4                   | Sticky Note                   | Notes AI response generation and sending | —                              | —                             | Generates and sends answer to Telegram                           |
| Sticky Note5                   | Sticky Note                   | Example input/output illustration   | —                              | —                             | 🟡 Example: Input "#vocabulary кукуруза" and output example     |
| Sticky Note6                   | Sticky Note                   | Notes typing emulation logic        | —                              | —                             | Checks for messages and emulates typing                          |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters:  
     - Updates: Select only `message`  
     - Credentials: Configure Telegram Bot token (from BotFather)  
   - Position: Entry point  
   - Connect output to "Show Typing Indicator" node.

2. **Create Telegram node ("Show Typing Indicator")**  
   - Type: Telegram  
   - Operation: `sendChatAction`  
   - Parameters:  
     - Chat ID: Expression: `{{ $('Start: Receive Message on Telegram').item.json.message.chat.id }}`  
     - Chat Action: `typing`  
   - Connect output to "Route by Input Type" switch node.

3. **Create Switch node ("Route by Input Type")**  
   - Type: Switch  
   - Parameters: Create 4 outputs with rules:  
     - Output 1 "Vocabulary": message text starts with `#vocabulary`  
     - Output 2 "Grammar": message text starts with `#grammar`  
     - Output 3 "Quiz": message text starts with `#quiz`  
     - Output 4 "Any other": fallback if message text exists but no hashtag matched  
   - Use expression for each condition:  
     `{{ $('Start: Receive Message on Telegram').item.json.message.text }}`  
   - Connect each output to its respective prompt node.

4. **Create Set node ("Vocabulary prompt")**  
   - Type: Set  
   - Mode: Raw  
   - JSON Output:  
     ```json
     {
       "prompt": "You are a Russian language tutor focused on expanding the student's vocabulary. The student will specify how many words they want to learn. Provide exactly that number of Russian words with English translations. Show pronunciation in Latin transcription. Use each word in a simple Russian sentence with English translation. At the end, give a small interactive task where the student can practice using some of the new words. create good looking formatting for telegram. use ** for bold text, instead ###. #vocabulary"
     }
     ```
   - Connect output to "Generate personalised answer".

5. **Create Set node ("Grammar prompt")**  
   - Type: Set  
   - Mode: Raw  
   - JSON Output:  
     ```json
     {
       "prompt": "You are a Russian grammar tutor. The student may ask about a specific grammar rule or request an explanation of a topic. Provide a clear explanation in English with Russian examples. Adjust the depth and number of examples based on the student's request. Ask the student to practice by forming their own sentences. Provide gentle corrections and explanations. create good looking formatting for telegram.  use ** for bold text, instead ###. #grammar"
     }
     ```
   - Connect output to "Generate personalised answer".

6. **Create Set node ("Quiz prompt")**  
   - Type: Set  
   - Mode: Raw  
   - JSON Output:  
     ```json
     {
       "prompt": "You are a Russian language tutor creating quizzes or exercises. The student decides how many questions they want. Format all tasks in a quiz-friendly style (multiple-choice, translation, or fill-in-the-blank). After the student answers, provide correct solutions and short explanations. End with positive feedback in Russian. create good looking formatting for telegram.  use ** for bold text, instead ###.#quiz"
     }
     ```
   - Connect output to "Generate personalised answer".

7. **Create Set node ("Main prompt for all other messages")**  
   - Type: Set  
   - Mode: Raw  
   - JSON Output:  
     ```json
     {
       "prompt": "You are an AI Russian language tutor. The user can choose a mode using hashtags: #vocabulary, #grammar, #quiz. Adapt your response format depending on the mode. The user decides how many items (words, rules, or questions) they want. Your task is to structure the response in the correct style, not to set limits. Always explain clearly in English with Russian examples. Always give feedback and encouragement in Russian at the end. Modes: #vocabulary — teach the requested number of words with pronunciation, examples, and a task. #grammar — explain the requested grammar topic with examples and practice. #quiz — create exactly the number of quiz questions requested, formatted properly, then give answers and feedback. If no hashtag is provided, ask the student to choose a mode. create good looking formatting for telegram. use ** for bold text, instead ###."
     }
     ```
   - Connect output to "Generate personalised answer".

8. **Create AIML API node ("Generate personalised answer")**  
   - Type: AIML API (OpenAI GPT-4o)  
   - Parameters:  
     - Model: `openai/gpt-4o`  
     - Prompt: Expression:  
       ```
       {{$json.prompt}}

       Message:
       {{ $('Start: Receive Message on Telegram').item.json.message.text }}
       ```  
   - Connect outputs from all prompt set nodes here.  
   - Connect output to "Send message to Telegram".  
   - Credentials: Configure OpenAI API key.

9. **Create Telegram node ("Send message to Telegram")**  
   - Type: Telegram  
   - Operation: Send Message  
   - Parameters:  
     - Text: Expression: `{{ $json.content }}`  
     - Chat ID: Expression: `{{ $('Start: Receive Message on Telegram').item.json.message.chat.id }}`  
     - Additional Fields: Disable append attribution  
   - Connect output from "Generate personalised answer".

10. **Activate the workflow**  
    - Verify all credentials are configured (Telegram bot token, OpenAI API key).  
    - Turn on the workflow and test via Telegram chat by sending messages tagged with #vocabulary, #grammar, #quiz, or plain text.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                              |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| 📖 Getting Started in 4 Steps: Setup Telegram bot token, AI API key, activate workflow, chat with hashtags, typing indicator enhances UX. | Sticky Note near workflow start, helps new users quickly deploy and test.                                   |
| 🔍 Node Overview: Telegram Trigger listens for messages; Typing Indicator improves UX; Router directs flow; Prompts customize AI input; Sends replies to Telegram. | Sticky Note summarizing node roles, useful for understanding workflow structure at a glance.                |
| 🟡 Example Input/Output: Input `#vocabulary кукуруза` produces vocabulary teaching with word, pronunciation, example sentence. | Example usage illustrating expected user input and AI output formatting.                                     |
| Telegram Bot Token Setup: Must be created via BotFather in Telegram app and inserted into Telegram nodes credential. | Essential step for enabling Telegram integration.                                                           |
| OpenAI API Key: Required for GPT-4o model access to generate AI responses.                          | Must be configured in n8n credentials for AI API node.                                                      |
| Use of **bold** formatting preferred over ### headings for Telegram message formatting.            | Ensures better visual appeal and compatibility with Telegram message rendering.                             |
| Workflow handles user input case-sensitively and expects hashtags at message start for routing.    | Important for users to know to place hashtags at the beginning to trigger correct mode.                      |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.