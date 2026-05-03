Log meal nutrients from Telegram to Google Sheets using an AI agent

https://n8nworkflows.xyz/workflows/log-meal-nutrients-from-telegram-to-google-sheets-using-an-ai-agent-3599


# Log meal nutrients from Telegram to Google Sheets using an AI agent

### 1. Workflow Overview

This workflow automates the logging of meal nutritional data sent via Telegram messages (text or voice) into a Google Sheet, leveraging AI for transcription and nutrient extraction. It targets users interested in nutrition tracking, such as health-conscious individuals, fitness coaches, or developers building health-related applications. The workflow is structured into the following logical blocks:

- **1.1 Input Reception:** Receives Telegram messages (text or voice) from users describing their meals.
- **1.2 Audio Processing:** If the input is a voice message, downloads and transcribes the audio using OpenAI.
- **1.3 AI Nutrient Extraction:** Sends the meal description (text or transcribed) to an OpenAI-powered agent to extract structured nutritional information.
- **1.4 Data Parsing and Storage:** Parses the AI response into individual nutrient entries, adds a timestamp, and stores the data in Google Sheets.
- **1.5 User Feedback:** Sends a confirmation message back to the user on Telegram.
- **1.6 Testing and Setup Notes:** Provides testing capabilities and setup instructions via sticky notes for ease of use and customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Captures incoming Telegram messages, distinguishing between text and voice inputs to route processing accordingly.
- **Nodes Involved:**  
  - Receive Telegram message  
  - If it's a voice message  
  - Set chatInput from message  
  - Set chatInput from voice  

- **Node Details:**

  - **Receive Telegram message**  
    - Type: Telegram Trigger  
    - Role: Listens for incoming Telegram messages or channel posts.  
    - Config: Monitors "message" and "channel_post" updates.  
    - Inputs: Telegram webhook trigger.  
    - Outputs: Passes message JSON to the next node.  
    - Failures: Possible webhook misconfiguration or Telegram API downtime.

  - **If it's a voice message**  
    - Type: If node (conditional branching)  
    - Role: Checks if the incoming message contains a voice object.  
    - Config: Condition tests existence of `$json.message.voice`.  
    - Inputs: Output from Telegram message node.  
    - Outputs: Two branches — true (voice message) and false (text message).  
    - Edge Cases: Messages without text or voice; unexpected message formats.

  - **Set chatInput from message**  
    - Type: Set node  
    - Role: Extracts text message content into a variable `chatInput`.  
    - Config: Assigns `$json.message.text` to `chatInput`.  
    - Inputs: False branch of voice message check.  
    - Outputs: Passes `chatInput` for AI processing.

  - **Set chatInput from voice**  
    - Type: Set node  
    - Role: Assigns transcribed text from voice to `chatInput`.  
    - Config: Uses `$json.text` (transcription result) as `chatInput`.  
    - Inputs: Output from transcription node.  
    - Outputs: Passes `chatInput` for AI processing.

---

#### 1.2 Audio Processing

- **Overview:** Downloads voice message audio files from Telegram and transcribes them into text using OpenAI.
- **Nodes Involved:**  
  - Get Audio File  
  - Transcribe Recording  

- **Node Details:**

  - **Get Audio File**  
    - Type: Telegram node (file resource)  
    - Role: Downloads the voice message audio file using Telegram's file ID.  
    - Config: Uses expression `{{$json.message.voice.file_id}}` to specify file.  
    - Inputs: True branch from voice message check.  
    - Outputs: Binary audio file data for transcription.  
    - Failures: Invalid file ID, Telegram API errors, network issues.

  - **Transcribe Recording**  
    - Type: OpenAI Audio node  
    - Role: Transcribes audio binary data into text.  
    - Config: Operation set to "transcribe", binary property name set to `data`.  
    - Inputs: Audio file binary from Get Audio File node.  
    - Outputs: JSON with transcription text in `$json.text`.  
    - Failures: Audio format unsupported, OpenAI API errors, timeouts.

---

#### 1.3 AI Nutrient Extraction

- **Overview:** Uses an OpenAI-powered agent to analyze the meal description text and extract a structured list of ingredients and nutritional values.
- **Nodes Involved:**  
  - When chat message received (Langchain chat trigger)  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - List of Ingredients and nutrients  

- **Node Details:**

  - **When chat message received**  
    - Type: Langchain Chat Trigger  
    - Role: Enables testing interface for chat input simulation.  
    - Config: Default webhook ID, no special parameters.  
    - Inputs: External test chat messages.  
    - Outputs: Passes input to AI agent.  
    - Notes: Useful for development and debugging.

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Sends prompt to OpenAI GPT-4o-mini model for processing.  
    - Config: Temperature set to 0 for deterministic output.  
    - Credentials: OpenAI API key configured.  
    - Inputs: Chat input text.  
    - Outputs: Raw AI response for parsing.  
    - Failures: API quota exceeded, invalid credentials, network issues.

  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses AI response into a JSON structure matching a nutrient schema.  
    - Config: Uses a JSON schema example with nutrient names, quantities, units, and reasoning.  
    - Inputs: AI raw response.  
    - Outputs: Structured JSON array of nutrient data.  
    - Failures: Parsing errors if AI output deviates from schema.

  - **List of Ingredients and nutrients**  
    - Type: Langchain Agent  
    - Role: Combines prompt, system message, and output parser to extract nutrient data.  
    - Config: Prompt requests approximate kcal, protein, carbs, lipids, electrolytes with breakdown and totals in JSON format; system message defines AI as nutrition expert.  
    - Inputs: `chatInput` string from previous nodes.  
    - Outputs: Parsed nutrient list JSON.  
    - Notes: Prompt is customizable to improve accuracy.

---

#### 1.4 Data Parsing and Storage

- **Overview:** Splits the nutrient list into individual entries, adds a date field, and appends each entry as a row in Google Sheets.
- **Nodes Involved:**  
  - Explode the list  
  - Add date  
  - Store in sheet  
  - Limit  

- **Node Details:**

  - **Explode the list**  
    - Type: SplitOut node  
    - Role: Splits the array of nutrient objects into individual items for processing.  
    - Config: Splits on the `output` field from AI agent.  
    - Inputs: Nutrient list JSON array.  
    - Outputs: One item per nutrient entry.  
    - Failures: Empty or malformed arrays.

  - **Add date**  
    - Type: Code node (JavaScript)  
    - Role: Adds a date field `my_date` to each nutrient entry in Excel serial date format.  
    - Config: Converts current date to Excel date number (days since 1899-12-30).  
    - Inputs: Individual nutrient JSON objects.  
    - Outputs: Nutrient entries with date added.  
    - Failures: Date conversion errors unlikely.

  - **Store in sheet**  
    - Type: Google Sheets node  
    - Role: Appends nutrient data rows to a specified Google Sheet.  
    - Config: Auto-maps fields `name`, `quantity`, `unit`, and `my_date` to sheet columns.  
    - Credentials: Google Sheets OAuth2 configured.  
    - Inputs: Nutrient entries with date.  
    - Outputs: Confirmation of append operation.  
    - Failures: Credential expiration, sheet access errors, quota limits.

  - **Limit**  
    - Type: Limit node  
    - Role: Controls flow after storage, here used to limit output to one item (likely for Telegram response).  
    - Inputs: Output from Google Sheets node.  
    - Outputs: Passes limited data downstream.  
    - Failures: None expected.

---

#### 1.5 User Feedback

- **Overview:** Sends a confirmation message back to the user on Telegram after successful logging.
- **Nodes Involved:**  
  - Respond message  

- **Node Details:**

  - **Respond message**  
    - Type: Telegram node (send message)  
    - Role: Sends a confirmation text "Your meal has been saved" to the user chat.  
    - Config: Uses chat ID from the original Telegram message JSON.  
    - Credentials: Telegram API token configured.  
    - Inputs: Output from Limit node.  
    - Outputs: Confirmation of message sent.  
    - Failures: Invalid chat ID, Telegram API errors.

---

#### 1.6 Testing and Setup Notes

- **Overview:** Provides user guidance, setup instructions, and workflow context via sticky notes.
- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4  
  - Sticky Note5  
  - Sticky Note6  
  - Sticky Note7  

- **Node Details:**

  - **Sticky Note**  
    - Content: Explains sending Telegram messages (text or voice) to log meals.  
  - **Sticky Note1**  
    - Content: Setup instructions including Telegram bot creation, Google Sheets setup, and OpenAI credential configuration.  
    - Link: [Telegram credential setup](https://docs.n8n.io/integrations/builtin/credentials/telegram/)  
  - **Sticky Note2**  
    - Content: Encourages users to check stored data for correctness and nutrient totals.  
  - **Sticky Note3**  
    - Content: Notes that audio files are transcribed using OpenAI.  
  - **Sticky Note4**  
    - Content: Testing instructions to chat with the workflow via the Langchain chat trigger.  
  - **Sticky Note5**  
    - Content: Suggests personalizing the AI prompt for better results.  
  - **Sticky Note6**  
    - Content: Suggests customizing the Telegram response message for politeness or detail.  
  - **Sticky Note7**  
    - Content: Contact information for workflow support and mention of a "Pro" version with USDA database integration.  
    - Contact: thomas@pollup.net

---

### 3. Summary Table

| Node Name                  | Node Type                             | Functional Role                                  | Input Node(s)                  | Output Node(s)                 | Sticky Note                                                                                          |
|----------------------------|-------------------------------------|-------------------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------------------------------------------------------|
| Receive Telegram message    | Telegram Trigger                    | Receives Telegram messages                       | —                             | If it's a voice message        | Setup instructions including Telegram bot creation, Google Sheets, OpenAI credentials (Sticky Note1) |
| If it's a voice message     | If node                            | Checks if message contains voice                 | Receive Telegram message       | Get Audio File, Set chatInput from message |                                                                                                    |
| Get Audio File              | Telegram (file resource)            | Downloads voice audio file                        | If it's a voice message        | Transcribe Recording           | Audio files transcribed using OpenAI (Sticky Note3)                                                |
| Transcribe Recording        | OpenAI Audio node                   | Transcribes audio to text                         | Get Audio File                 | Set chatInput from voice       |                                                                                                    |
| Set chatInput from voice    | Set node                          | Sets transcribed text as chatInput               | Transcribe Recording           | List of Ingredients and nutrients |                                                                                                    |
| Set chatInput from message  | Set node                          | Sets text message as chatInput                    | If it's a voice message        | List of Ingredients and nutrients |                                                                                                    |
| When chat message received  | Langchain Chat Trigger             | Testing interface for chat input                  | —                             | List of Ingredients and nutrients | Testing instructions for chat simulation (Sticky Note4)                                           |
| OpenAI Chat Model           | Langchain OpenAI Chat Model        | Sends prompt to OpenAI for nutrient extraction   | When chat message received     | List of Ingredients and nutrients | Personalize the AI prompt (Sticky Note5)                                                           |
| Structured Output Parser    | Langchain Structured Output Parser | Parses AI response into structured JSON           | OpenAI Chat Model              | List of Ingredients and nutrients |                                                                                                    |
| List of Ingredients and nutrients | Langchain Agent                   | Extracts nutrient data from chatInput             | Set chatInput from voice, Set chatInput from message, When chat message received, OpenAI Chat Model, Structured Output Parser | Explode the list                | Personalize prompt and response message (Sticky Note5, Sticky Note6)                              |
| Explode the list            | SplitOut node                     | Splits nutrient list into individual entries      | List of Ingredients and nutrients | Add date                      |                                                                                                    |
| Add date                   | Code node (JavaScript)             | Adds date field in Excel format                    | Explode the list              | Store in sheet                 |                                                                                                    |
| Store in sheet             | Google Sheets node                 | Appends nutrient data to Google Sheet             | Add date                     | Limit                         | Check your data for correctness (Sticky Note2)                                                    |
| Limit                     | Limit node                        | Limits output flow                                 | Store in sheet               | Respond message               | Personalize response message (Sticky Note6)                                                       |
| Respond message            | Telegram node (send message)       | Sends confirmation message to Telegram user       | Limit                        | —                             |                                                                                                    |
| Sticky Note                | Sticky Note                      | Explains Telegram message input                    | —                             | —                             | Send Telegram message instructions                                                                |
| Sticky Note1               | Sticky Note                      | Setup instructions                                 | —                             | —                             | Setup instructions with link to Telegram docs                                                     |
| Sticky Note2               | Sticky Note                      | Data checking advice                               | —                             | —                             | Check your data                                                                                   |
| Sticky Note3               | Sticky Note                      | Audio transcription note                           | —                             | —                             | Audio transcription using OpenAI                                                                  |
| Sticky Note4               | Sticky Note                      | Testing instructions                               | —                             | —                             | Testing instructions                                                                              |
| Sticky Note5               | Sticky Note                      | Prompt personalization advice                      | —                             | —                             | Personalize the prompt                                                                            |
| Sticky Note6               | Sticky Note                      | Response message customization advice              | —                             | —                             | Personalize response message                                                                     |
| Sticky Note7               | Sticky Note                      | Contact and pro version info                        | —                             | —                             | Contact info and pro version mention                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Bot and Credentials**  
   - Use BotFather to create a Telegram bot and obtain the API token.  
   - In n8n, create Telegram API credentials with this token.

2. **Create Google Sheet and Credentials**  
   - Create an empty Google Sheet with columns: `name`, `quantity`, `unit`, `my_date`.  
   - Note the Google Sheet document ID and sheet name (default `gid=0`).  
   - In n8n, create Google Sheets OAuth2 credentials with access to this sheet.

3. **Create OpenAI Credentials**  
   - Obtain OpenAI API key.  
   - Configure OpenAI credentials in n8n.

4. **Add Telegram Trigger Node**  
   - Node: Telegram Trigger  
   - Configure to listen for updates: `message` and `channel_post`.  
   - Use Telegram API credentials.

5. **Add If Node to Check for Voice Message**  
   - Condition: Check if `$json.message.voice` exists.  
   - True branch: voice message processing.  
   - False branch: text message processing.

6. **Voice Message Processing Branch**  
   - Add Telegram node (file resource) to download audio file using `{{$json.message.voice.file_id}}`.  
   - Add OpenAI Audio node to transcribe audio: set operation to "transcribe", binary property to `data`. Use OpenAI credentials.  
   - Add Set node to assign transcription text (`$json.text`) to variable `chatInput`.

7. **Text Message Processing Branch**  
   - Add Set node to assign text message content (`$json.message.text`) to variable `chatInput`.

8. **Add Langchain Chat Trigger Node (Optional for Testing)**  
   - Allows manual chat input simulation.

9. **Add OpenAI Chat Model Node**  
   - Model: `gpt-4o-mini` (or preferred GPT-4 variant).  
   - Temperature: 0 for deterministic output.  
   - Use OpenAI credentials.

10. **Add Structured Output Parser Node**  
    - Configure with JSON schema example for nutrients: name, quantity, unit, reasoning.

11. **Add Langchain Agent Node (List of Ingredients and nutrients)**  
    - Prompt:  
      ```
      Approximate the kcals, protein, carbohydrates, lipids (fats), and electrolyte (sodium, potassium, magnesium, Zinc and Iron) content in the following dietary intake statement:  
      {{ $json.chatInput }}  
      Provide estimates for each component based on typical nutritional values. Break down the contributions from each food item (steak, salad, vinaigrette) and give a total number for each nutrient.  
      Give the total result as a json, with as name, the name of the nutrient, as quantity, the total summed value, and as unit the unit that been chosen (gr, mg).  
      Put the reasoning in another variable called "reasonning"
      ```  
    - System message: "You are a nutrition expert."  
    - Enable output parser with the structured output parser node.

12. **Connect branches from Set nodes and Langchain Chat Trigger to the Agent node.**

13. **Add SplitOut node (Explode the list)**  
    - Split the `output` array from the agent into individual nutrient entries.

14. **Add Code node (Add date)**  
    - JavaScript code to add `my_date` field as Excel serial date number (days since 1899-12-30).

15. **Add Google Sheets node (Store in sheet)**  
    - Operation: Append  
    - Sheet Name: `gid=0` or your sheet name  
    - Document ID: your Google Sheet ID  
    - Map columns: `name`, `quantity`, `unit`, `my_date`  
    - Use Google Sheets OAuth2 credentials.

16. **Add Limit node**  
    - Limits output to one item (for Telegram response).

17. **Add Telegram node (Respond message)**  
    - Text: "Your meal has been saved"  
    - Chat ID: `{{$json.message.chat.id}}` from original Telegram message  
    - Use Telegram API credentials.

18. **Connect nodes in order:**  
    - Telegram Trigger → If node → (voice branch) Get Audio File → Transcribe Recording → Set chatInput from voice → Agent node  
    - (text branch) Set chatInput from message → Agent node  
    - Agent node → Explode the list → Add date → Store in sheet → Limit → Respond message

19. **Add Sticky Notes for documentation and user guidance as desired.**

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                     |
|---------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Setup instructions include creating Telegram bot, Google Sheet, and OpenAI credentials.                       | [Telegram credential setup](https://docs.n8n.io/integrations/builtin/credentials/telegram/)       |
| Testing interface available via Langchain Chat Trigger node for simulating inputs and debugging.              | Testing instructions in Sticky Note4                                                              |
| Prompt customization recommended to improve nutrient extraction accuracy.                                     | Sticky Note5                                                                                       |
| Response message can be personalized for politeness or additional feedback.                                   | Sticky Note6                                                                                       |
| Contact for workflow modifications, help, or custom workflows: thomas@pollup.net                              | Sticky Note7                                                                                       |
| A "Pro" version exists with USDA database integration for detailed nutrient breakdowns.                       | Mentioned in Sticky Note7                                                                          |
| Date is stored in Excel serial date format for compatibility with Google Sheets date functions.               | Code node "Add date"                                                                                |

---

This document provides a complete, structured reference to understand, reproduce, and customize the "Log meal nutrients from Telegram to Google Sheets using an AI agent" workflow. It covers all nodes, logic blocks, configurations, and potential failure points to facilitate robust usage and development.