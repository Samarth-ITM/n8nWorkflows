Automated Reservation System with Telegram, Google Gemini AI, and Google Sheets

https://n8nworkflows.xyz/workflows/automated-reservation-system-with-telegram--google-gemini-ai--and-google-sheets-5454


# Automated Reservation System with Telegram, Google Gemini AI, and Google Sheets

### 1. Workflow Overview

This workflow automates court reservation requests received via Telegram, leveraging Google Gemini AI for natural language understanding, Google Sheets for data storage and availability checks, and Gmail for sending confirmation emails. It is designed for a sports club to streamline booking of training courts through a single-message interaction in Telegram.

**Use cases:**  
- Users submit a reservation request in one Telegram message including date, name, email, court number, start time, and end time.  
- The workflow validates input, checks availability against existing bookings, confirms or suggests alternatives, stores the reservation, sends email confirmation, and replies back to the user in Telegram.

**Logical blocks:**  
- **1.1 Input Reception:** Captures raw Telegram messages.  
- **1.2 Prompt Preparation:** Formats user input with date and context for AI processing.  
- **1.3 AI Processing & Memory:** Uses Google Gemini AI and Langchain agent with Postgres memory to parse, validate, and manage dialog.  
- **1.4 Court Availability Check:** Queries Google Sheets for existing court bookings and availability.  
- **1.5 Reservation Storage:** Writes confirmed bookings to Google Sheets.  
- **1.6 Email Confirmation:** Sends the reservation confirmation email via Gmail.  
- **1.7 Telegram Reply:** Sends a confirmation or error message back to the user.  
- **1.8 Auxiliary:** Sticky note for documentation and overview.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Listens for incoming Telegram messages from users requesting court reservations.  
- **Nodes:** Telegram Trigger  

**Node: Telegram Trigger**  
- Type: Telegram Trigger node (webhook-based)  
- Configuration: Watches for "message" updates only. Uses Telegram API credentials.  
- Key variables: Incoming `message.text` and `message.chat.id` for user identification and reply target.  
- Inputs: External Telegram webhook call.  
- Outputs: Passes raw Telegram message JSON downstream.  
- Edge cases: Network issues, Telegram API limits, malformed messages.  
- Version: 1.2  

---

#### 1.2 Prompt Preparation

- **Overview:** Extracts message text and formats a prompt adding today’s date for AI context.  
- **Nodes:** Prepare Prompt  

**Node: Prepare Prompt**  
- Type: AI Transform node with custom JS code  
- Configuration: Runs JavaScript to format today’s date as "Month Day, Year", extracts Telegram message text, constructs `finalPrompt` string.  
- Key expressions: Uses `$input.all()` to gather all incoming items and builds prompt with embedded date and user question.  
- Inputs: Message JSON from Telegram Trigger.  
- Outputs: Returns an object with `finalPrompt` field for AI agent input.  
- Edge cases: Missing or corrupted message text, date formatting errors.  
- Version: 1  

---

#### 1.3 AI Processing & Memory

- **Overview:** Uses Google Gemini language model and Langchain agent to interpret user input, maintain session memory, validate requests, and orchestrate reservation logic.  
- **Nodes:** Google Gemini Chat Model, AI Agent, Postgres Chat Memory  

**Node: Google Gemini Chat Model**  
- Type: Google Gemini language model node  
- Configuration: Uses model "models/gemini-2.5-flash-preview-04-17" with Google PaLM API credentials.  
- Inputs: `finalPrompt` from Prepare Prompt node.  
- Outputs: AI-generated text responses.  
- Edge cases: API quota, latency, parsing errors, credential failures.  
- Version: 1  

**Node: AI Agent**  
- Type: Langchain Agent Node  
- Configuration: Uses a detailed system message defining court reservation assistant behavior, validation rules, error handling, and tone. It receives `finalPrompt` as input text and uses Google Gemini as language model and Postgres for chat memory.  
- Key expressions: Uses $json.finalPrompt for input.  
- Inputs: Prompt from Prepare Prompt, AI language model output, and chat memory.  
- Outputs: Final AI response text for Telegram reply and extracted structured data for downstream nodes.  
- Edge cases: Misinterpretation, unexpected user input, memory retrieval errors, timeouts.  
- Version: 1.9  

**Node: Postgres Chat Memory**  
- Type: Langchain Postgres Chat Memory  
- Configuration: Stores conversational context per Telegram chat ID as session key, with a context window of 20 messages. Uses Postgres credentials.  
- Inputs: Conversation context from Telegram chat.  
- Outputs: Provides context to AI Agent.  
- Edge cases: Database connectivity, session key collisions, data truncation.  
- Version: 1.3  

---

#### 1.4 Court Availability Check

- **Overview:** Retrieves current court reservation data from Google Sheets to check availability and conflicts.  
- **Nodes:** court info (Google Sheets Tool)  

**Node: court info**  
- Type: Google Sheets Tool  
- Configuration: Reads from a Google Sheets document (court info sheet) with court availability or resource info, using Google OAuth2 credentials.  
- Key parameters: Document ID and sheet name fixed to a specific Google Sheets URL and tab.  
- Inputs: Triggered by AI Agent to check availability.  
- Outputs: Returns sheet data for AI to validate requested reservation times.  
- Edge cases: Sheet access errors, permission issues, data format inconsistencies.  
- Version: 4.6  

---

#### 1.5 Reservation Storage

- **Overview:** Appends or updates confirmed reservation records in Google Sheets.  
- **Nodes:** Google Sheets (court confirmations)  

**Node: Google Sheets**  
- Type: Google Sheets Tool  
- Configuration: Appends or updates reservation records in a confirmation sheet with columns Date, name, email, court, start time, end time, and confirmed status. Uses Google OAuth2 credentials.  
- Key expressions: Fields are mapped dynamically from AI extracted data variables (e.g., Date, name, email) using AI override expressions.  
- Inputs: Data from AI Agent after validation and confirmation.  
- Outputs: Updates the spreadsheet with the new reservation.  
- Edge cases: Sheet write failures, concurrency conflicts, data validation errors.  
- Version: 4.6  

---

#### 1.6 Email Confirmation

- **Overview:** Sends an email confirmation to the user with reservation details.  
- **Nodes:** Gmail  

**Node: Gmail**  
- Type: Gmail node (send email)  
- Configuration: Sends email with dynamic To, Subject, and Message fields populated from AI agent output. Uses Gmail OAuth2 credentials.  
- Key expressions: `sendTo`, `subject`, and `message` are populated from AI extracted variables using override expressions.  
- Inputs: AI Agent outputs structured email content.  
- Outputs: Sends email and returns status.  
- Edge cases: Gmail API rate limits, auth token expiration, malformed email addresses.  
- Version: 2.1  

---

#### 1.7 Telegram Reply

- **Overview:** Sends the final confirmation or error message back to the user on Telegram.  
- **Nodes:** Telegram  

**Node: Telegram**  
- Type: Telegram node (send message)  
- Configuration: Sends text from AI Agent output back to the originating Telegram chat using chat ID from Telegram Trigger.  
- Inputs: Output text from AI Agent.  
- Outputs: Confirmation message sent to Telegram user.  
- Edge cases: Telegram API failures, chat ID missing, message too long.  
- Version: 1.2  

---

#### 1.8 Auxiliary

- **Overview:** Provides detailed documentation and notes about the workflow for maintainers or users.  
- **Nodes:** Sticky Note  

**Node: Sticky Note**  
- Type: Documentation sticky note  
- Content: Explains workflow purpose, input data collected, logic flow, Google Sheets setup, notes on data headers, and contact email for support.  
- Position: Visual documentation only, no inputs or outputs.  

---

### 3. Summary Table

| Node Name           | Node Type                       | Functional Role                       | Input Node(s)       | Output Node(s)      | Sticky Note                                                                                                            |
|---------------------|---------------------------------|-------------------------------------|---------------------|---------------------|------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger     | Telegram Trigger                | Input reception from Telegram       | -                   | Prepare Prompt      | 🧠 Purpose: powers general-purpose reservation bot using Telegram, Sheets, Email; captures one-message booking request. |
| Prepare Prompt      | AI Transform (JS code)          | Formats prompt with date and input  | Telegram Trigger    | AI Agent            | Same as above                                                                                                           |
| Google Gemini Chat Model | Google Gemini AI model         | Language model to process prompt    | Prepare Prompt      | AI Agent            | Same as above                                                                                                           |
| Postgres Chat Memory | Langchain Postgres Chat Memory  | Stores session chat context         | Telegram Trigger    | AI Agent            | Same as above                                                                                                           |
| AI Agent            | Langchain Agent                 | Orchestrates dialog and validation  | Prepare Prompt, Google Gemini, Court info, Postgres Chat Memory, Google Sheets, Gmail | Telegram            | Same as above                                                                                                           |
| court info          | Google Sheets Tool              | Reads current court info from Sheets| AI Agent            | AI Agent            | Same as above                                                                                                           |
| Google Sheets       | Google Sheets Tool              | Stores confirmed bookings           | AI Agent            | AI Agent            | Same as above                                                                                                           |
| Gmail               | Gmail                          | Sends confirmation emails           | AI Agent            | -                   | Same as above                                                                                                           |
| Telegram            | Telegram                       | Sends reply to user in Telegram     | AI Agent            | -                   | Same as above                                                                                                           |
| Sticky Note         | Sticky Note                    | Documentation                       | -                   | -                   | Workflow purpose, logic description, input fields, Google Sheets setup, support contact.                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates only  
   - Credentials: Connect your Telegram Bot API credentials  
   - Position: Input start of workflow  

2. **Create Prepare Prompt node**  
   - Type: AI Transform node (JS code)  
   - JS Code: Extract current date formatted like "Month Day, Year" and Telegram message text to build `finalPrompt` string:  
     ```js
     const items = $input.all();
     const today = new Date();
     const formattedDate = `${today.toLocaleString("default", { month: "long" })} ${today.getDate()}, ${today.getFullYear()}`;
     return items.map((item) => {
       const telegramMessage = item?.json?.message?.text;
       return { finalPrompt: `Today's date is: ${formattedDate}\n\nUser's question:\n${telegramMessage}` };
     });
     ```
   - Connect input from Telegram Trigger  
   - Output: finalPrompt for AI processing  

3. **Add Google Gemini Chat Model node**  
   - Type: Google Gemini language model node  
   - Parameters:  
     - Model name: "models/gemini-2.5-flash-preview-04-17"  
     - Options: default  
   - Credentials: Google Palm API credentials  
   - Connect input from Prepare Prompt  

4. **Add Postgres Chat Memory node**  
   - Type: Langchain Postgres Chat Memory  
   - Parameters:  
     - Session Key: set to Telegram chat ID from Telegram Trigger (expression: `={{ $('Telegram Trigger').item.json.message.chat.id }}`)  
     - Context window length: 20  
   - Credentials: Postgres database connection  
   - Connect input from Telegram Trigger (for session key)  

5. **Create AI Agent node**  
   - Type: Langchain Agent node  
   - Parameters:  
     - Text input: `={{ $json.finalPrompt }}` from Prepare Prompt  
     - System message: Copy the detailed assistant instructions for court booking (including greeting, data collection, validation, error handling, tone) as provided.  
     - Prompt type: Define  
     - Link AI language model: Google Gemini Chat Model  
     - Link chat memory: Postgres Chat Memory  
   - Connect inputs from Prepare Prompt, Google Gemini Chat Model, Postgres Chat Memory  

6. **Create court info node (Google Sheets Tool)**  
   - Type: Google Sheets Tool (read)  
   - Parameters:  
     - Document ID: Google Sheets document containing court availability info  
     - Sheet name: appropriate sheet/tab (e.g., "Sheet1")  
   - Credentials: Google Sheets OAuth2 credentials  
   - Connect input from AI Agent (ai_tool input)  

7. **Create Google Sheets node for reservation storage**  
   - Type: Google Sheets Tool (append or update)  
   - Parameters:  
     - Document ID: Google Sheets document for court confirmations  
     - Sheet name: e.g., "Sheet1"  
     - Columns: map Date, name, email, court, start time, end time, confirmed fields dynamically via expressions from AI Agent outputs  
     - Matching Columns: Date (for updates)  
   - Credentials: Google Sheets OAuth2 credentials  
   - Connect input from AI Agent (ai_tool input)  

8. **Create Gmail node**  
   - Type: Gmail (send email)  
   - Parameters:  
     - Send To: email variable from AI Agent output  
     - Subject: from AI Agent output  
     - Message: from AI Agent output  
     - Email Type: text  
   - Credentials: Gmail OAuth2 credentials  
   - Connect input from AI Agent (ai_tool input)  

9. **Create Telegram node (send message)**  
   - Type: Telegram (send message)  
   - Parameters:  
     - Text: output text from AI Agent (`={{ $json.output }}`)  
     - Chat ID: from Telegram Trigger (`={{ $('Telegram Trigger').item.json.message.chat.id }}`)  
   - Credentials: Telegram API credentials  
   - Connect input from AI Agent (main output)  

10. **Create Sticky Note node for documentation**  
    - Type: Sticky Note  
    - Content: Paste the detailed workflow description, input fields explained, main logic flow, Google Sheets setup instructions, and contact email for support.  
    - Position as desired, no connections needed.  

11. **Configure workflow execution order as per connections above.**  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                           | Context or Link                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| 🧠 This workflow powers a general-purpose reservation bot integrating Telegram, Google Sheets, and Gmail with AI assistance. Designed for Black Ball Sporting Club but adaptable to other reservation systems.                                                                                                       | Sticky Note content in the workflow                                                                     |
| ⚠️ Google Sheets must have two spreadsheets: one for court info (availability) and one for reservation confirmations with columns: Date, Name, Email, Court, Start Time, End Time, Confirmed.                                                                                                                            | Sticky Note and Google Sheets node configuration                                                      |
| 📬 For support customizing or deploying the bot, contact: tharwat.elsayed.hamad@gmail.com                                                                                                                                                                                                                              | Sticky Note content                                                                                     |
| The AI Agent’s system message includes detailed behavior, data collection, validation rules, error messages, and tone guidelines to ensure smooth user interactions and error handling.                                                                                                                               | AI Agent node configuration                                                                            |
| Use Google Gemini AI and Langchain with Postgres chat memory to maintain conversational context and improve user experience.                                                                                                                                                                                            | Nodes: Google Gemini Chat Model, AI Agent, Postgres Chat Memory                                       |
| Gmail node uses OAuth2 credentials for secure email sending. Ensure tokens are refreshed properly to avoid sending failures.                                                                                                                                                                                             | Gmail node configuration                                                                               |
| Telegram Trigger and Telegram nodes require proper Telegram Bot API credentials and webhook setup to receive and send messages.                                                                                                                                                                                         | Telegram nodes configuration                                                                           |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, adhering strictly to content policies without any illegal, offensive, or protected material. All data handled is legal and public.