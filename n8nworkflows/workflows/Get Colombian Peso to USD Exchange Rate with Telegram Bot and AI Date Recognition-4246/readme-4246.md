Get Colombian Peso to USD Exchange Rate with Telegram Bot and AI Date Recognition

https://n8nworkflows.xyz/workflows/get-colombian-peso-to-usd-exchange-rate-with-telegram-bot-and-ai-date-recognition-4246


# Get Colombian Peso to USD Exchange Rate with Telegram Bot and AI Date Recognition

### 1. Workflow Overview

This workflow enables a Telegram bot to provide the Colombian Peso to USD exchange rate (TRM - Tasa Representativa del Mercado) for a user-specified date. It intelligently processes user input in text or audio form, uses AI to extract and normalize the date, validates the date against the current local Colombian time, and queries the Colombian government open data API for the exchange rate. If data is unavailable for the requested date, it searches up to 10 previous days to find the most recent valid rate. The workflow then responds back to the user via Telegram with the exchange rate or an appropriate message if no data is found.

#### Logical Blocks:

- **1.1 Input Reception and Type Validation**  
  Handles reception of Telegram messages (text or voice), distinguishes input type, and prepares text for processing.

- **1.2 Audio Transcription**  
  Converts voice messages to text via OpenAI’s transcription.

- **1.3 Date Extraction with AI**  
  Uses OpenAI-powered agents to extract and normalize the date from user input text.

- **1.4 Date Validation**  
  Compares extracted date against current local Colombian date; blocks future dates.

- **1.5 TRM Data Retrieval**  
  Queries the official Colombian government API for exchange rate data on the requested date.

- **1.6 Data Availability and Fallback Loop**  
  If data for the requested date is missing, loops back up to 10 days prior to find data.

- **1.7 Result Sorting and Final Response**  
  Sorts retrieved data by date, selects the most recent valid rate, and sends the final message to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Type Validation

- **Overview:**  
  Receives user messages from Telegram and determines if input is text or voice. Prepares text field accordingly.

- **Nodes Involved:**  
  - Once a Telegram Message is received  
  - Validate Text or Audio  
  - Text only  
  - Audio Text  
  - Edit Fields

- **Node Details:**

  - **Once a Telegram Message is received**  
    - Type: Telegram Trigger  
    - Role: Entry point; triggers workflow on new Telegram messages (text or voice).  
    - Config: Listens for "message" updates; uses Telegram API credentials (bot token).  
    - Inputs: Telegram webhook  
    - Outputs: Raw message JSON  

  - **Validate Text or Audio**  
    - Type: Switch  
    - Role: Branches flow based on input type: voice or text.  
    - Config: Checks existence of `message.voice` or `message.text` fields to route accordingly.  
    - Inputs: Telegram message JSON  
    - Outputs: Two branches, "voice" and "text"  

  - **Text only**  
    - Type: Set  
    - Role: Extracts `message.text` into a simplified `text` field for downstream processing.  
    - Config: Sets `text = message.text`.  
    - Inputs: Text branch from Switch  
    - Outputs: JSON containing `text`  

  - **Audio Text**  
    - Type: Set  
    - Role: Prepares transcribed audio text for further usage (similar to "Text only" node).  
    - Config: Sets `text` field from previous transcription output.  
    - Inputs: Transcribed text JSON  
    - Outputs: JSON containing `text`  

  - **Edit Fields**  
    - Type: Set  
    - Role: Passes the `text` field forward for AI processing without modification.  
    - Config: Sets `text` from input JSON’s `text`.  
    - Inputs: Both Text only and Audio Text nodes converge here.  
    - Outputs: JSON with `text` ready for AI agent  

- **Edge Cases / Failures:**  
  - Missing or malformed Telegram messages  
  - Voice message without file_id or corrupted files  
  - Text messages that are empty or non-informative  

---

#### 1.2 Audio Transcription

- **Overview:**  
  Downloads voice message files from Telegram and sends them to OpenAI for transcription into Spanish text.

- **Nodes Involved:**  
  - Download Audio  
  - Transcribe Audio

- **Node Details:**

  - **Download Audio**  
    - Type: Telegram node (file download)  
    - Role: Downloads voice message file from Telegram servers using file_id.  
    - Config: Uses Telegram API credentials; downloads the file as binary data.  
    - Inputs: Voice message branch from Switch  
    - Outputs: Binary audio file  

  - **Transcribe Audio**  
    - Type: OpenAI node (audio resource)  
    - Role: Sends audio binary to OpenAI Whisper model for transcription in Spanish.  
    - Config: Language set to "es", temperature 0 for deterministic output, property name set to binary data.  
    - Credentials: OpenAI API key required  
    - Inputs: Binary audio file from Download Audio  
    - Outputs: JSON with transcribed text  

- **Edge Cases / Failures:**  
  - File download failures (network, invalid file_id)  
  - OpenAI transcription timeouts or errors  
  - Audio quality issues leading to inaccurate transcription  

---

#### 1.3 Date Extraction with AI

- **Overview:**  
  Uses an OpenAI-powered agent to analyze the input text and extract a date in ISO format (YYYY-MM-DD), applying specific rules for Spanish date expressions.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - Extractor Agent

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Langchain OpenAI Chat Model  
    - Role: Core AI model (GPT-4 nano) for natural language understanding.  
    - Config: Model "gpt-4.1-nano", temperature 0, max retries 2.  
    - Credentials: OpenAI API  
    - Inputs: Text from Edit Fields  
    - Outputs: AI-generated chat responses  

  - **Structured Output Parser**  
    - Type: Langchain structured output parser  
    - Role: Parses AI output to extract a JSON object with a `date` field.  
    - Config: Expects JSON schema `{ "date": "YYYY-MM-DD" }`.  
    - Inputs: AI chat output  
    - Outputs: JSON with normalized date  

  - **Extractor Agent**  
    - Type: Langchain Agent  
    - Role: Coordinates input text, system prompt, and output parsing to reliably extract date.  
    - Config:  
      - System message defines expert algorithm rules to extract Spanish dates, handle ambiguities, and fallback to current date if none found.  
      - Input text is passed dynamically.  
    - Inputs: Text, local current date (for contextual reference)  
    - Outputs: JSON with extracted date field  

- **Edge Cases / Failures:**  
  - AI model failing to produce valid JSON output  
  - Ambiguous or no date found in input text  
  - Unexpected input formats or languages other than Spanish  

---

#### 1.4 Date Validation

- **Overview:**  
  Validates extracted date against current local Colombian date; rejects any date in the future by notifying the user.

- **Nodes Involved:**  
  - Local Current Date and Time  
  - Validate if Date is in the past  
  - Notify past date

- **Node Details:**

  - **Local Current Date and Time**  
    - Type: Code node  
    - Role: Computes the current date and datetime adjusted to Colombia timezone (UTC-5).  
    - Config: Outputs date string `fecha_actual` and datetime string `fecha_hora_actual` in ISO-like formats.  
    - Inputs: From Edit Fields  
    - Outputs: JSON with current local date/time  

  - **Validate if Date is in the past**  
    - Type: If node  
    - Role: Checks if extracted date is after current date (future date).  
    - Config: Condition type = dateTime, operator = "after", compares extracted date vs. local current date.  
    - Inputs: Extracted date and local date  
    - Outputs: Two branches: future date (true), valid date (false)  

  - **Notify past date**  
    - Type: Telegram node  
    - Role: Sends Telegram message to user notifying that the date is in the future and invalid.  
    - Config: Static text "Fecha posterior a fecha actual" (Date after current date), chatId from original Telegram message sender.  
    - Inputs: True branch of validation If node  
    - Outputs: Confirmation of message sent  

- **Edge Cases / Failures:**  
  - Timezone miscalculations  
  - Date format inconsistencies  
  - Telegram message delivery failures  

---

#### 1.5 TRM Data Retrieval

- **Overview:**  
  Queries the Colombian government open data API for the TRM exchange rate on the requested date.

- **Nodes Involved:**  
  - Get TRM  
  - Check if Valor exists  
  - Send message to user

- **Node Details:**

  - **Get TRM**  
    - Type: HTTP Request  
    - Role: Requests TRM data for the extracted date via API `https://www.datos.gov.co/resource/32sa-8pi3.json?vigenciadesde=YYYY-MM-DD`.  
    - Config: URL dynamically built with extracted date.  
    - Inputs: Valid date branch from validation node  
    - Outputs: JSON array of TRM data  

  - **Check if Valor exists**  
    - Type: If node  
    - Role: Checks if the retrieved data contains a non-empty `valor` field (exchange rate value).  
    - Config: Condition checks existence of `valor` string.  
    - Inputs: API response JSON  
    - Outputs: Two branches: data exists (true), no data (false)  

  - **Send message to user**  
    - Type: Telegram node  
    - Role: Sends the found exchange rate back to the user including the date.  
    - Config: Message text includes `valor` and date from Extractor Agent node, chatId from original sender.  
    - Inputs: True branch of Check if Valor exists  
    - Outputs: Confirmation message sent  

- **Edge Cases / Failures:**  
  - API downtime or rate limiting  
  - Malformed or empty API responses  
  - Telegram message delivery errors  

---

#### 1.6 Data Availability and Fallback Loop

- **Overview:**  
  If no data exists for the requested date, the workflow tries up to 10 previous days to find the latest available TRM.

- **Nodes Involved:**  
  - Generate an array with 10 numbers  
  - Split Items for the loop  
  - Get the last 10 responses  
  - Convert date  
  - Get TRM for past date  
  - Get non-empty rows  
  - If (check again for valor)  
  - Send no data  

- **Node Details:**

  - **Generate an array with 10 numbers**  
    - Type: Code node  
    - Role: Creates an array `[1..10]` to represent days to look back.  
    - Outputs: JSON with array `counter`  

  - **Split Items for the loop**  
    - Type: Split Out  
    - Role: Splits the array into individual items to process sequentially.  
    - Inputs: Output of the array generator  
    - Outputs: Individual day offsets  

  - **Get the last 10 responses**  
    - Type: Split In Batches  
    - Role: Controls batch processing of the 10 day-offset requests.  
    - Inputs: Split items  
    - Outputs: Batch items  

  - **Convert date**  
    - Type: Code node  
    - Role: Adjusts the extracted date by subtracting the counter number of days to get each fallback date.  
    - Logic: For each counter `n`, subtract `n` days from the extracted date; outputs `adjustedDate`.  
    - Inputs: Batch counter items and extracted date  
    - Outputs: JSON with adjusted dates and loop counter  

  - **Get TRM for past date**  
    - Type: HTTP Request  
    - Role: Queries API with each adjusted date for TRM data.  
    - Config: Similar to Get TRM node but uses `adjustedDate`.  
    - Inputs: Output of Convert date  
    - Outputs: API responses for each fallback date  

  - **Get non-empty rows**  
    - Type: Filter  
    - Role: Filters API responses to keep only those containing a `valor` field.  
    - Inputs: Batch of API responses  
    - Outputs: Responses with valid TRM values  

  - **If**  
    - Type: If node  
    - Role: Checks if any valid TRM data exists after filtering.  
    - Outputs:  
      - True: Proceed to sorting and sending the latest data  
      - False: Send no data message  

  - **Send no data**  
    - Type: Telegram node  
    - Role: Informs the user that no TRM data exists for the requested date or fallback dates.  
    - Config: Message includes the requested date for clarity.  
    - Inputs: False branch of If node  
    - Outputs: Confirmation message sent  

- **Sticky Note:**  
  "Reason for Loop: Some TRM are valid for several days, like during weekends or holidays, so 10 days are validated to grab the last available TRM."

- **Edge Cases / Failures:**  
  - API limits when calling multiple times  
  - Date adjustments crossing month/year boundaries  
  - No data found even after 10-day fallback  
  - Telegram message delivery failures  

---

#### 1.7 Result Sorting and Final Response

- **Overview:**  
  Sorts the collected valid TRM data by date descending, limits to the most recent entry, and sends the final exchange rate message to the user.

- **Nodes Involved:**  
  - Sort most recent data  
  - Get the last data  
  - Send current TRM

- **Node Details:**

  - **Sort most recent data**  
    - Type: Sort  
    - Role: Sorts TRM data items descending by `vigenciadesde` date field to find the newest.  
    - Inputs: Valid TRM data array  
    - Outputs: Sorted array  

  - **Get the last data**  
    - Type: Limit  
    - Role: Limits output to one item (the most recent TRM).  
    - Inputs: Sorted data  
    - Outputs: Single TRM item  

  - **Send current TRM**  
    - Type: Telegram node  
    - Role: Sends the final TRM value and date to the user.  
    - Config: Message text includes the `valor` and the date.  
    - Inputs: Limited latest TRM item  
    - Outputs: Confirmation message sent to Telegram user  

- **Edge Cases / Failures:**  
  - Sorting errors due to malformed dates  
  - No data reaching this stage if fallback fails  
  - Telegram message delivery errors  

---

### 3. Summary Table

| Node Name                      | Node Type                                | Functional Role                                      | Input Node(s)                        | Output Node(s)                         | Sticky Note                                                                                                                         |
|--------------------------------|-----------------------------------------|-----------------------------------------------------|------------------------------------|--------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Once a Telegram Message is received | Telegram Trigger                       | Entry point; receives Telegram messages             | -                                  | Validate Text or Audio                |                                                                                                                                    |
| Validate Text or Audio          | Switch                                  | Branches message type: text or voice                 | Once a Telegram Message is received | Download Audio, Text only             |                                                                                                                                    |
| Text only                      | Set                                     | Extracts text field from message                      | Validate Text or Audio               | Edit Fields                          |                                                                                                                                    |
| Audio Text                     | Set                                     | Prepares transcribed text                             | Transcribe Audio                    | Edit Fields                          |                                                                                                                                    |
| Edit Fields                   | Set                                     | Passes text for AI processing                         | Text only, Audio Text               | Local Current Date and Time           |                                                                                                                                    |
| Download Audio                 | Telegram node (file download)            | Downloads voice message audio                         | Validate Text or Audio              | Transcribe Audio                     |                                                                                                                                    |
| Transcribe Audio               | OpenAI (audio transcription)             | Transcribes audio to text                             | Download Audio                     | Audio Text                          |                                                                                                                                    |
| Local Current Date and Time    | Code                                    | Computes current date/time in Colombia timezone      | Edit Fields                       | Extractor Agent                      |                                                                                                                                    |
| OpenAI Chat Model              | Langchain OpenAI Chat Model              | AI model for natural language understanding          | Extractor Agent (via agent)        | Extractor Agent                     |                                                                                                                                    |
| Structured Output Parser       | Langchain Output Parser                  | Parses AI output JSON                                 | OpenAI Chat Model                  | Extractor Agent                     |                                                                                                                                    |
| Extractor Agent               | Langchain Agent                          | Extracts normalized date from text                    | Local Current Date and Time, Edit Fields | Validate if Date is in the past      |                                                                                                                                    |
| Validate if Date is in the past | If                                      | Checks if extracted date is after current date       | Extractor Agent                   | Notify past date, Get TRM             |                                                                                                                                    |
| Notify past date              | Telegram                                | Notifies user that date is invalid (future date)     | Validate if Date is in the past    | -                                    |                                                                                                                                    |
| Get TRM                      | HTTP Request                            | Queries TRM data for extracted date                   | Validate if Date is in the past    | Check if Valor exists                 |                                                                                                                                    |
| Check if Valor exists          | If                                      | Checks if API returned TRM data                        | Get TRM                          | Send message to user, Generate an array with 10 numbers |                                                                                                                                    |
| Send message to user          | Telegram                                | Sends TRM value and date to user                      | Check if Valor exists             | -                                    |                                                                                                                                    |
| Generate an array with 10 numbers | Code                                  | Creates array [1..10] for fallback date search       | Check if Valor exists (no data)    | Split Items for the loop              | Reason for Loop: Some TRM are valid for several days, like during weekends or holidays, so 10 days are validated to grab the last available TRM |
| Split Items for the loop      | Split Out                               | Splits array into individual items for looping        | Generate an array with 10 numbers  | Get the last 10 responses             |                                                                                                                                    |
| Get the last 10 responses     | Split In Batches                        | Processes the fallback dates in batches               | Split Items for the loop           | Get non-empty rows, Convert date      |                                                                                                                                    |
| Convert date                 | Code                                    | Adjusts extracted date by subtracting days for fallback | Get the last 10 responses          | Get TRM for past date                 |                                                                                                                                    |
| Get TRM for past date          | HTTP Request                           | Queries TRM API for each adjusted fallback date       | Convert date                      | Get the last 10 responses             |                                                                                                                                    |
| Get non-empty rows            | Filter                                  | Filters API responses to those with a valid `valor`   | Get the last 10 responses          | If                                 |                                                                                                                                    |
| If                           | If                                      | Checks if any valid TRM data exists                     | Get non-empty rows                 | Sort most recent data, Send no data   |                                                                                                                                    |
| Send no data                 | Telegram                                | Notifies user no TRM data found                         | If                              | -                                    |                                                                                                                                    |
| Sort most recent data         | Sort                                    | Sorts TRM data descending by date                       | If (valid data branch)             | Get the last data                    |                                                                                                                                    |
| Get the last data             | Limit                                   | Limits data to the most recent TRM                      | Sort most recent data              | Send current TRM                    |                                                                                                                                    |
| Send current TRM              | Telegram                                | Sends final TRM value and date to user                  | Get the last data                 | -                                    |                                                                                                                                    |
| Calculator                   | Langchain Calculator Tool                | Part of AI agent tooling (unused in main flow)          | OpenAI Chat Model                 | Extractor Agent                     |                                                                                                                                    |
| Think                        | Langchain Think Tool                     | Part of AI agent tooling (unused in main flow)          | OpenAI Chat Model                 | Extractor Agent                     |                                                                                                                                    |
| Sticky Note                  | Sticky Note                             | Explains workflow purpose and flow                      | -                                | -                                    | # 📌 Workflow: Request TRM (Representative Market Rate)\n\n## 🎯 Objective\nRetrieve the TRM for a user date via Telegram\n\n## 🔗 Data Source\nhttps://www.datos.gov.co/ |
| Sticky Note1                 | Sticky Note                             | Workflow overview and description                        | -                                | -                                    | # 📌 Workflow: Request TRM (Representative Market Rate)\n\n## 🎯 Objective\nRetrieve the TRM (Colombian Peso to US Dollar exchange rate) for a specific date provided by the user via Telegram. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Parameters: Listen for `message` updates  
   - Credentials: Connect Telegram Bot API credentials (OAuth2 or token)  
   - Position: Start of workflow  

2. **Add Switch Node "Validate Text or Audio"**  
   - Type: Switch  
   - Rules:  
     - Output "voice" if `message.voice` exists  
     - Output "text" if `message.text` exists  
   - Connect input from Telegram Trigger  

3. **Create "Download Audio" Node** (Voice branch)  
   - Type: Telegram (file download)  
   - Parameters: File ID = `message.voice.file_id`  
   - Credentials: Same Telegram credentials  
   - Connect input from Switch voice output  

4. **Create "Transcribe Audio" Node**  
   - Type: OpenAI node (audio resource)  
   - Parameters:  
     - Language: Spanish (`es`)  
     - Temperature: 0  
     - Binary Property Name: `data` (from Download Audio)  
   - Credentials: OpenAI API key  
   - Connect input from Download Audio output  

5. **Create "Audio Text" Set Node**  
   - Type: Set  
   - Parameter: Set field `text` = transcription text from Transcribe Audio node  
   - Connect input from Transcribe Audio output  

6. **Create "Text only" Set Node** (Text branch)  
   - Type: Set  
   - Parameter: Set field `text` = `message.text`  
   - Connect input from Switch text output  

7. **Create "Edit Fields" Set Node**  
   - Type: Set  
   - Parameter: Set field `text` = input `text` (from either Audio Text or Text only)  
   - Connect inputs from both Audio Text and Text only nodes  

8. **Create "Local Current Date and Time" Code Node**  
   - Type: Code  
   - Code:  
     ```javascript
     const now = new Date();
     const utc = now.getTime() + now.getTimezoneOffset() * 60000;
     const colombiaTime = new Date(utc - 5 * 3600000);
     const fecha = colombiaTime.toISOString().split('T')[0];
     const fechaHora = colombiaTime.toISOString().replace('T', ' ').substring(0, 19);
     return [{ json: { fecha_actual: fecha, fecha_hora_actual: fechaHora } }];
     ```  
   - Connect input from Edit Fields  

9. **Create "Extractor Agent" Langchain Agent Node**  
   - Type: Langchain Agent  
   - Parameters:  
     - Text input: `text` from Edit Fields  
     - System message (prompt): Explain date extraction rules in Spanish (as per original)  
     - Output parser: Structured Output Parser (next node)  
   - Connect input from Local Current Date and Time  

10. **Create "Structured Output Parser" Node**  
    - Type: Langchain Output Parser  
    - Parameters: JSON schema expecting `{ "date": "YYYY-MM-DD" }`  
    - Connect input from OpenAI Chat Model used by Extractor Agent  

11. **Create "Validate if Date is in the past" If Node**  
    - Type: If  
    - Condition: Check if extracted date (`output.date`) is after local current date (`fecha_actual`)  
    - Connect input from Extractor Agent output  

12. **Create "Notify past date" Telegram Node**  
    - Type: Telegram  
    - Text: "Fecha posterior a fecha actual"  
    - Chat ID: User's Telegram ID from original message  
    - Connect input from "true" branch of If node (future date)  

13. **Create "Get TRM" HTTP Request Node**  
    - Type: HTTP Request  
    - URL: `https://www.datos.gov.co/resource/32sa-8pi3.json?vigenciadesde={{ $json.output.date }}`  
    - Connect input from "false" branch of If node (valid date)  

14. **Create "Check if Valor exists" If Node**  
    - Type: If  
    - Condition: Check if `valor` field exists and is not empty in API response  
    - Connect input from Get TRM node  

15. **Create "Send message to user" Telegram Node**  
    - Type: Telegram  
    - Text: `{{ $json.valor }} TRM del {{ extracted date }}`  
    - Chat ID: User's Telegram ID  
    - Connect input from "true" branch of Check if Valor exists  

16. **Create "Generate an array with 10 numbers" Code Node**  
    - Type: Code  
    - Code: Create array `[1..10]` for fallback days  
    - Connect input from "false" branch of Check if Valor exists  

17. **Create "Split Items for the loop" Node**  
    - Type: Split Out  
    - Field: `counter` (array element)  
    - Connect input from Generate an array node  

18. **Create "Get the last 10 responses" Split In Batches Node**  
    - Type: Split In Batches  
    - Connect input from Split Items for the loop  

19. **Create "Convert date" Code Node**  
    - Type: Code  
    - Code: Adjust extracted date minus counter days, output `adjustedDate`  
    - Connect input from Get the last 10 responses  

20. **Create "Get TRM for past date" HTTP Request Node**  
    - Type: HTTP Request  
    - URL: `https://www.datos.gov.co/resource/32sa-8pi3.json?vigenciadesde={{ $json.adjustedDate }}`  
    - Connect input from Convert date  

21. **Create "Get non-empty rows" Filter Node**  
    - Type: Filter  
    - Condition: Check existence of `valor` field in responses  
    - Connect input from Get the last 10 responses  

22. **Create "If" Node**  
    - Type: If  
    - Condition: Check if filtered rows contain any data  
    - Connect input from Get non-empty rows  

23. **Create "Send no data" Telegram Node**  
    - Type: Telegram  
    - Text: "No existe información para la fecha proporcionada: {{ extracted date }}"  
    - Chat ID: User's Telegram ID  
    - Connect input from "false" branch of If node  

24. **Create "Sort most recent data" Sort Node**  
    - Type: Sort  
    - Sort by: `vigenciadesde` descending  
    - Connect input from "true" branch of If node  

25. **Create "Get the last data" Limit Node**  
    - Type: Limit  
    - Limit: 1  
    - Connect input from Sort node  

26. **Create "Send current TRM" Telegram Node**  
    - Type: Telegram  
    - Text: `{{ $json.valor }} TRM del {{ extracted date }}`  
    - Chat ID: User's Telegram ID  
    - Connect input from Get the last data node  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                        | Context or Link                                             |
|-----------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Workflow retrieves Colombian Peso to USD exchange rate (TRM) for a user-provided date via Telegram bot, handling text and voice input with AI date extraction and fallback queries. | Workflow purpose summary                                    |
| Data source: Open data portal of Colombian government — TRM dataset: https://www.datos.gov.co/resource/32sa-8pi3.json                               | TRM data API documentation                                  |
| AI date extraction prompt includes rules for ambiguous Spanish date expressions, fallback rules, and always outputs date in ISO format YYYY-MM-DD. | Prompt design for Extractor Agent                           |
| Loop fallback searches last 10 days prior to requested date because TRM data may cover multiple days (weekends/holidays).                         | Sticky note reason for fallback loop                         |
| Telegram Bot credentials require OAuth2/token setup; OpenAI API key required for chat and transcription nodes.                                      | Credential setup notes                                      |
| Timezone is adjusted manually to Colombia time (UTC-5) inside a code node to ensure date validation accuracy.                                        | Local date/time handling                                    |
| For reliable AI output parsing, the structured output parser expects a strict JSON with a single `date` field.                                     | AI output parser configuration                               |

---

**Disclaimer:** This document is generated from an automated n8n workflow export and adheres to all content policies. It contains no illegal or protected data.