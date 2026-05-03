Generate Language Learning Flashcards with GPT-4, Telegram and Google Sheets for Anki

https://n8nworkflows.xyz/workflows/generate-language-learning-flashcards-with-gpt-4--telegram-and-google-sheets-for-anki-7199


# Generate Language Learning Flashcards with GPT-4, Telegram and Google Sheets for Anki

### 1. Workflow Overview

This workflow automates the generation of language learning flashcards by leveraging GPT-4 via OpenAI, Telegram for user input and output interaction, and Google Sheets for storage. It is intended for users who want to quickly create flashcards for language learning, particularly to import into Anki or similar spaced repetition tools.

The logic is divided into the following functional blocks:

- **1.1 Input Reception:** Captures user input from Telegram messages or manual triggers.
- **1.2 AI Processing:** Uses an AI agent powered by GPT-4 to generate translations and example sentences formatted for flashcards.
- **1.3 Data Storage:** Appends the generated flashcard data into a Google Sheet.
- **1.4 Confirmation:** Sends a confirmation message back to the user on Telegram with the flashcard content.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures the input flashcard front text either via Telegram messages or a manual workflow trigger for testing purposes. It initializes the flashcard creation process.

**Nodes Involved:**  
- Telegram Trigger  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Flashcard 'front' (Set node)  
- Sticky Note (Input flashcard 'front')

**Node Details:**  

- **Telegram Trigger**  
  - Type: Trigger (Telegram)  
  - Role: Listens for incoming Telegram messages from users.  
  - Configuration: Watches for "message" updates, with download enabled.  
  - Credentials: Uses a Telegram bot account.  
  - Output: Passes message JSON including text and chat details.  
  - Potential Failures: Telegram API downtime, invalid bot token, or message format issues.

- **When clicking ‘Execute workflow’**  
  - Type: Manual Trigger  
  - Role: Allows manual execution for testing by injecting a preset input.  
  - Configuration: No parameters; triggers downstream nodes.  
  - Output: Triggers the workflow with default test data.

- **Flashcard 'front'**  
  - Type: Set node  
  - Role: Assigns a test input value "comprehensive" to simulate a user's flashcard front text.  
  - Configuration: Sets variable `message.text` to "comprehensive".  
  - Input: Receives trigger from manual node.  
  - Output: Passes the set data downstream.

- **Sticky Note**  
  - Content: "### Input flashcard 'front' Telegram or manual"  
  - Role: Documentation within the workflow for clarity about input sources.

---

#### 2.2 AI Processing

**Overview:**  
This block takes the input text and invokes an AI agent based on GPT-4 to generate the flashcard back content (translation and example sentences) in a structured JSON format.

**Nodes Involved:**  
- AI Agent1 (Langchain Agent)  
- OpenAI Chat Model1  
- Structured Output Parser1  
- Sticky Note1 (Generate flashcard 'back')

**Node Details:**  

- **AI Agent1**  
  - Type: Langchain Agent Node  
  - Role: Core AI processing node that sends user input to GPT-4 and receives structured responses.  
  - Configuration:  
    - Input: Passes the Telegram or manual input text via expression `={{ $json.message.text }}`.  
    - System Message: Defines the agent's role as a translation assistant generating JSON output with fields: word, translation, sentences.  
    - Output Parser: Enabled to parse structured JSON output from AI.  
  - Input: Receives input text from Telegram Trigger or Flashcard 'front'.  
  - Output: Outputs parsed JSON with flashcard data.  
  - Edge Cases: Possible failures include incomplete or malformed AI responses, API rate limits, or network issues.

- **OpenAI Chat Model1**  
  - Type: Language Model (OpenAI GPT-4)  
  - Role: Provides the GPT-4 model backend for AI Agent1.  
  - Configuration: Uses the "gpt-4o-mini" model variant with default options.  
  - Credentials: Requires valid OpenAI API credentials.  
  - Input/Output: Connected internally to AI Agent1 as its language model provider.

- **Structured Output Parser1**  
  - Type: Output Parser  
  - Role: Converts AI raw text output into structured JSON based on defined schema: word, translation, sentences.  
  - Configuration: Uses a JSON schema example to parse outputs properly.  
  - Input/Output: Connected within AI Agent1’s parser pipeline.

- **Sticky Note1**  
  - Content: "### Generate flashcard 'back'"  
  - Role: Clarifies the purpose of this block within the workflow.

---

#### 2.3 Data Storage

**Overview:**  
This block appends the generated flashcard information (front and back) into a Google Sheet, representing the flashcard deck which can later be imported into Anki.

**Nodes Involved:**  
- Append row in sheet (Google Sheets)  
- Sticky Note2 (Store flashcard in Sheet)

**Node Details:**  

- **Append row in sheet**  
  - Type: Google Sheets node  
  - Role: Adds a new row to a specified Google Sheet with flashcard question and answer columns.  
  - Configuration:  
    - Operation: Append row.  
    - Document ID: Points to specific Google Sheets document "TomekAnkiPersonalDeck".  
    - Sheet Name: Uses sheet identified by gid=0.  
    - Columns Mapped:  
      - question ← output.word (flashcard front)  
      - answer ← output.translation + `<br/><br/>` + output.sentences (flashcard back)  
    - Credentials: Google Sheets OAuth2 credentials required.  
  - Input: Receives flashcard JSON from AI Agent1.  
  - Output: Passes success data downstream.  
  - Edge Cases: Failures may include authorization errors, Google Sheets API limits, or invalid/missing document ID.

- **Sticky Note2**  
  - Content: "### Store flashcard in Sheet"  
  - Role: Describes the purpose of this node.

---

#### 2.4 Confirmation

**Overview:**  
After successfully storing the flashcard, this block sends a confirmation message back to the user on Telegram containing the generated flashcard content.

**Nodes Involved:**  
- Send a text message (Telegram)  
- Sticky Note3 (Confirm)

**Node Details:**  

- **Send a text message**  
  - Type: Telegram node (send message)  
  - Role: Sends a message back to the Telegram user confirming the flashcard creation.  
  - Configuration:  
    - Text: Combines `question` and `answer` fields from the Google Sheets append output to format the confirmation message.  
    - Chat ID: Extracted dynamically via expression `={{ $json.chat.id }}` to reply to the correct user.  
    - Additional Fields: Attribution disabled for cleaner message.  
  - Credentials: Uses the same Telegram bot credentials as Telegram Trigger.  
  - Input: Receives data from Google Sheets node.  
  - Edge Cases: Failures could arise from invalid chat IDs, Telegram API errors, or network issues.

- **Sticky Note3**  
  - Content: "### Confirm"  
  - Role: Indicates this is the confirmation step for the user.

---

### 3. Summary Table

| Node Name               | Node Type                       | Functional Role                      | Input Node(s)                  | Output Node(s)              | Sticky Note                                              |
|-------------------------|--------------------------------|------------------------------------|-------------------------------|-----------------------------|----------------------------------------------------------|
| Telegram Trigger         | Telegram Trigger                | Input reception via Telegram        | —                             | AI Agent1                   | Input flashcard 'front' Telegram or manual               |
| When clicking ‘Execute workflow’ | Manual Trigger                 | Manual test trigger                 | —                             | Flashcard 'front'            | Input flashcard 'front' Telegram or manual               |
| Flashcard 'front'        | Set                            | Sets test input value               | When clicking ‘Execute workflow’ | AI Agent1                   | Input flashcard 'front' Telegram or manual               |
| AI Agent1               | Langchain Agent                | AI processing for flashcard back   | Telegram Trigger, Flashcard 'front' | Append row in sheet          | Generate flashcard 'back'                                 |
| OpenAI Chat Model1       | Langchain OpenAI GPT-4 Model   | Provides GPT-4 model backend        | AI Agent1 (ai_languageModel)  | AI Agent1                   |                                                          |
| Structured Output Parser1 | Langchain Output Parser         | Parses AI output JSON               | AI Agent1 (ai_outputParser)   | AI Agent1                   |                                                          |
| Append row in sheet      | Google Sheets                  | Stores flashcard in Google Sheet    | AI Agent1                     | Send a text message          | Store flashcard in Sheet                                  |
| Send a text message      | Telegram                       | Confirms flashcard creation to user | Append row in sheet           | —                           | Confirm                                                  |
| Sticky Note              | Sticky Note                    | Documentation                      | —                             | —                           | Input flashcard 'front' Telegram or manual               |
| Sticky Note1             | Sticky Note                    | Documentation                      | —                             | —                           | Generate flashcard 'back'                                |
| Sticky Note2             | Sticky Note                    | Documentation                      | —                             | —                           | Store flashcard in Sheet                                  |
| Sticky Note3             | Sticky Note                    | Documentation                      | —                             | —                           | Confirm                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credential**  
   - Register a Telegram bot via BotFather.  
   - In n8n, create Telegram API credentials with the bot token.

2. **Create Google Sheets OAuth2 Credential**  
   - Set up OAuth2 credentials for Google Sheets with proper scopes to edit sheets.

3. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node named "When clicking ‘Execute workflow’".

4. **Create Set Node for Test Input**  
   - Add a "Set" node named "Flashcard 'front'".  
   - Configure to assign `message.text` with string value "comprehensive".  
   - Connect "When clicking ‘Execute workflow’" to "Flashcard 'front'".

5. **Create Telegram Trigger Node**  
   - Add a "Telegram Trigger" node named "Telegram Trigger".  
   - Configure to listen for "message" updates with download enabled.  
   - Attach the Telegram bot credentials.

6. **Create Langchain OpenAI Chat Model Node**  
   - Add a "Langchain OpenAI Chat Model" node named "OpenAI Chat Model1".  
   - Select model "gpt-4o-mini".  
   - Attach OpenAI API credentials.

7. **Create Structured Output Parser Node**  
   - Add a "Langchain Output Parser Structured" node named "Structured Output Parser1".  
   - Paste JSON schema example for fields: word, translation, sentences.

8. **Create Langchain Agent Node**  
   - Add a "Langchain Agent" node named "AI Agent1".  
   - Configure input text as expression `={{ $json.message.text }}`.  
   - Set system message to instruct generating translations and sentences in JSON, as per the role and context described in the overview.  
   - Enable output parser and assign "Structured Output Parser1" for parsing.  
   - Connect "OpenAI Chat Model1" to "AI Agent1" as the language model.  
   - Connect "Structured Output Parser1" to "AI Agent1" as the output parser.  
   - Connect both "Telegram Trigger" and "Flashcard 'front'" nodes to "AI Agent1" main input.

9. **Create Google Sheets Node**  
   - Add a "Google Sheets" node named "Append row in sheet".  
   - Configure operation "Append".  
   - Specify Google Sheets document ID (your Anki deck sheet) and sheet gid (usually 0).  
   - Map columns:  
     - question ← `{{$json.output.word}}`  
     - answer ← `{{$json.output.translation}}<br/><br/>{{$json.output.sentences}}`  
   - Attach Google Sheets OAuth2 credentials.  
   - Connect "AI Agent1" to "Append row in sheet".

10. **Create Telegram Send Message Node**  
    - Add a "Telegram" node named "Send a text message".  
    - Set message text to:  
      ```
      {{$json.question}}

      {{$json.answer}}
      ```  
    - Set chat ID to `{{$json.chat.id}}`.  
    - Disable message attribution.  
    - Attach Telegram credentials.  
    - Connect "Append row in sheet" to "Send a text message".

11. **Add Sticky Notes**  
    - Add sticky notes for documentation at logical points: input reception, AI processing, storage, confirmation, using the content provided.

12. **Verify Connections**  
    - Telegram Trigger → AI Agent1  
    - Flashcard 'front' → AI Agent1  
    - OpenAI Chat Model1 → AI Agent1 (ai_languageModel)  
    - Structured Output Parser1 → AI Agent1 (ai_outputParser)  
    - AI Agent1 → Append row in sheet  
    - Append row in sheet → Send a text message  
    - Manual Trigger → Flashcard 'front'

13. **Test Workflow**  
    - Execute manually via the manual trigger node or send a Telegram message.  
    - Confirm that flashcards are appended in the Google Sheet and confirmation is sent on Telegram.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| The output JSON from GPT-4 must strictly follow the schema with fields "word", "translation", and "sentences" to ensure proper parsing.    | Critical for Structured Output Parser node functioning                                              |
| Use `<br>` tags in translations and examples to format multi-line flashcard content compatible with Anki HTML card fields.                  | Formatting convention used in AI prompt system message                                              |
| Workflow designed for personal or small group use; Google Sheets API and OpenAI API limits may apply at scale.                             | Consider API quotas when scaling                                                                    |
| Telegram bot token and Google Sheets OAuth2 credentials must have appropriate permissions for message reading and sheet editing respectively. | Credential setup critical for operation                                                             |
| For more information on n8n Langchain nodes and structured output parsers, visit: https://docs.n8n.io/integrations/builtin/nodes/langchain/ | Documentation link for advanced customization                                                       |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.