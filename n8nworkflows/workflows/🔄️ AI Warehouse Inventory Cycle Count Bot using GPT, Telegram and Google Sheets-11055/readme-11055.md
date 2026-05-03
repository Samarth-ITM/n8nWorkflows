🔄️ AI Warehouse Inventory Cycle Count Bot using GPT, Telegram and Google Sheets

https://n8nworkflows.xyz/workflows/----ai-warehouse-inventory-cycle-count-bot-using-gpt--telegram-and-google-sheets-11055


# 🔄️ AI Warehouse Inventory Cycle Count Bot using GPT, Telegram and Google Sheets

### 1. Workflow Overview

This workflow implements an **AI-powered Inventory Cycle Count Bot** for warehouse management using Telegram, OpenAI GPT models, and Google Sheets. It is designed to assist warehouse operators in conducting inventory cycle counts through voice commands sent via Telegram. The bot transcribes voice messages, extracts structured inventory data (location ID and counted quantity), validates the inputs, updates a Google Sheet inventory record accordingly, and guides the operator through remaining locations until completion.

The workflow is logically divided into the following blocks:

- **1.1 Telegram Input and Command Handling:** Receives Telegram updates, distinguishes between commands and voice messages, and handles command logic.
- **1.2 Audio Processing and AI Extraction:** Downloads voice messages, transcribes audio to text, then extracts structured information (location ID and quantity) using AI language models.
- **1.3 Validation and Error Handling:** Checks transcription completeness and location validity, asks for repeats or sends error messages as needed.
- **1.4 Inventory Update:** Updates the Google Sheets inventory record with the actual counted quantity.
- **1.5 Cycle Count Progression:** Retrieves remaining locations to be counted, sends instructions for next or last location, and controls workflow looping until all locations are processed.

---

### 2. Block-by-Block Analysis

#### 2.1 Telegram Input and Command Handling

- **Overview:**  
  This block captures Telegram messages, determines if the message is a recognized command (/start, /help) or a voice message, and routes accordingly.

- **Nodes Involved:**  
  - Telegram Trigger  
  - Is Audio? (If node)  
  - Command (If node)  
  - Command Selection (Switch node)  
  - Help Message (Telegram node)  
  - Wrong Command (Telegram node)  

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Entry point; listens to Telegram messages (updates of type "message").  
    - Config: Listens for all incoming messages.  
    - Output: Sends message JSON downstream.  
    - Edge cases: Webhook failure, Telegram API rate limits.

  - **Is Audio?**  
    - Type: If node  
    - Role: Checks if incoming message contains a voice file (voice.file_id exists).  
    - Config: Condition tests if `message.voice.file_id` exists; routes voice messages right, others left.  
    - Edge cases: Missing file_id, malformed messages.

  - **Command**  
    - Type: If node  
    - Role: Checks if text message matches recognized commands (/start or /help).  
    - Config: Tests if message.text is in ['/start', '/help'].  
    - Edge cases: Case sensitivity, unexpected text formats.

  - **Command Selection**  
    - Type: Switch node  
    - Role: Routes recognized commands specifically to /start or /help paths.  
    - Config: Conditions for exact matches to /start or /help.  
    - Edge cases: Commands other than the two, empty or malformed text.

  - **Help Message**  
    - Type: Telegram node  
    - Role: Sends a help message explaining how to use the bot.  
    - Config: Markdown-formatted detailed instructions, sent to chat ID from trigger.  
    - Edge cases: Telegram API errors, chatId missing.

  - **Wrong Command**  
    - Type: Telegram node  
    - Role: Sends a warning message when an unrecognized command or message is received.  
    - Config: Text message prompting valid commands, sent to chat ID.  
    - Edge cases: Similar to Help Message.

---

#### 2.2 Audio Processing and AI Extraction

- **Overview:**  
  This block handles voice message processing: downloads audio, transcribes it with AI, then extracts structured JSON data (`location_id` and `quantity`) using GPT-4.

- **Nodes Involved:**  
  - Collect Audio (Telegram node)  
  - Transcribe Operator Command (OpenAI audio node)  
  - Model: Information Extraction (OpenAI GPT chat)  
  - Structured Output Parser (LangChain output parser)  
  - Get Location, Qty (LangChain agent)  

- **Node Details:**

  - **Collect Audio**  
    - Type: Telegram node  
    - Role: Downloads the voice file from Telegram using `voice.file_id`.  
    - Config: Uses Telegram API credentials; dynamic fileId from incoming message.  
    - Edge cases: File download failure, Telegram API limits.

  - **Transcribe Operator Command**  
    - Type: OpenAI audio transcription node  
    - Role: Converts audio binary to text transcription.  
    - Config: Uses OpenAI STT model; input from audio binary property named "data".  
    - Edge cases: Transcription failures, API timeouts, unsupported audio format.

  - **Model: Information Extraction**  
    - Type: OpenAI GPT chat node  
    - Role: Performs natural language understanding to extract inventory info.  
    - Config: GPT-4.1-mini model, no additional options.  
    - Input: Raw transcription text.  
    - Output: Chat completion with extracted info.  
    - Edge cases: Model latency, incomplete or ambiguous text.

  - **Structured Output Parser**  
    - Type: LangChain output parser  
    - Role: Enforces output to match JSON schema with `location_id` and `quantity`.  
    - Config: JSON schema example ensures output is structured.  
    - Edge cases: Parsing errors, unexpected model outputs.

  - **Get Location, Qty**  
    - Type: LangChain agent node  
    - Role: Combines model and parser to produce validated structured output.  
    - Config: System prompt explicitly instructs JSON-only response with nulls for missing data.  
    - Edge cases: Null values returned, ambiguous transcription.

---

#### 2.3 Validation and Error Handling

- **Overview:**  
  Validates extracted data; if `location_id` or `quantity` is missing (null), requests operator to repeat. Also verifies location existence and whether location is already counted.

- **Nodes Involved:**  
  - Transcription Error (If node)  
  - Ask to Repeat (Telegram node)  
  - Collect Remaining Locations (Google Sheets read)  
  - Combine with Location Records (Merge node)  
  - Merge Transcript With Location List (Merge node)  
  - Is Location in Scope? (If node)  
  - Error Message: Wrong Location (Telegram node)  

- **Node Details:**

  - **Transcription Error**  
    - Type: If node  
    - Role: Checks if either `location_id` or `quantity` is null in AI output.  
    - Config: Conditions check if these fields equal string "null".  
    - Output: Yes routes to "Ask to repeat", No continues processing.  
    - Edge cases: False negatives if string "null" not properly normalized.

  - **Ask to Repeat**  
    - Type: Telegram node  
    - Role: Sends a message requesting operator to repeat voice message clearly.  
    - Config: Text message explains problem; sent to chat ID.  
    - Edge cases: Telegram delivery failures.

  - **Collect Remaining Locations**  
    - Type: Google Sheets node  
    - Role: Reads all locations from the inventory sheet where `checked` ≠ "X" (unchecked locations).  
    - Config: Filters rows with `checked` != "X", sheet and doc IDs configured.  
    - Edge cases: Google API quota, missing rows.

  - **Combine with Location Records**  
    - Type: Merge node (combineBySql)  
    - Role: Combines newly transcribed location data with Google Sheets rows to validate location presence.  
    - Config: SQL style join; no specific options.  
    - Edge cases: Null merges, no matching location.

  - **Merge Transcript With Location List**  
    - Type: Merge node (combineBySql)  
    - Role: Similar to above, merges transcript data with location list to check scope.  
    - Edge cases: Same as above.

  - **Is Location in Scope?**  
    - Type: If node  
    - Role: Checks if extracted `location_id` exists in the list of remaining locations.  
    - Config: Condition tests if `location_ids` array includes the extracted location.  
    - Output: Yes proceeds to update inventory; No sends error message.  
    - Edge cases: Location not found due to casing or format mismatch.

  - **Error Message: Wrong Location**  
    - Type: Telegram node  
    - Role: Sends an error message when location is invalid or already counted.  
    - Config: Text warns operator; chat ID from trigger.  
    - Edge cases: Telegram API errors.

---

#### 2.4 Inventory Update

- **Overview:**  
  Updates the Google Sheet inventory record with the actual counted quantity and marks the location as checked.

- **Nodes Involved:**  
  - Update Inventory Quantity (Google Sheets node)  
  - Wait (Wait node)  

- **Node Details:**

  - **Update Inventory Quantity**  
    - Type: Google Sheets node  
    - Role: Updates row matching `location_id` with `actual_quantity` and sets `checked` = "V".  
    - Config: Mapping defines these fields; sheet and document ID specified.  
    - Edge cases: Update conflicts, API limits, row not found.

  - **Wait**  
    - Type: Wait node  
    - Role: Placeholder for potential delay or pacing between updates. Currently default no delay.  
    - Edge cases: None unless delay configured.

---

#### 2.5 Cycle Count Progression

- **Overview:**  
  Determines remaining locations, sends instructions for next or last location, and controls workflow continuation.

- **Nodes Involved:**  
  - Get Locations to Check (Google Sheets read)  
  - Extract Location IDs (Aggregate node)  
  - Remaining Locations (Aggregate node)  
  - Number of Locations (Set node)  
  - Last Location ? (If node)  
  - Send Instruction: Next Location (Telegram node)  
  - Send Instruction: Last Location (Telegram node)  
  - First Location (Limit node)  

- **Node Details:**

  - **Get Locations to Check**  
    - Type: Google Sheets node  
    - Role: Reads all unchecked locations from sheet (filter `checked` = "X").  
    - Config: Document and sheet ID specified.  
    - Edge cases: Empty results, quota errors.

  - **Extract Location IDs**  
    - Type: Aggregate node  
    - Role: Aggregates all `location_id` fields into a list (`location_ids`).  
    - Config: Single field aggregation.  
    - Edge cases: Empty input.

  - **Remaining Locations**  
    - Type: Aggregate node  
    - Role: Counts number of remaining unchecked locations.  
    - Config: Aggregates length of location_ids array.  
    - Edge cases: None.

  - **Number of Locations**  
    - Type: Set node  
    - Role: Stores count of remaining locations in variable `remaining_locations`.  
    - Edge cases: None.

  - **Last Location ?**  
    - Type: If node  
    - Role: Checks if only one remaining location left.  
    - Config: Compares `remaining_locations` == 1.  
    - Output: Yes routes to send last location instruction; No routes to send next location instruction.  
    - Edge cases: Off-by-one errors.

  - **Send Instruction: Next Location**  
    - Type: Telegram node  
    - Role: Sends message instructing operator to go to next location and count units with voice message.  
    - Config: Message includes variable location_id.  
    - Edge cases: Telegram delivery failures.

  - **Send Instruction: Last Location**  
    - Type: Telegram node  
    - Role: Sends final location instruction with markdown formatting.  
    - Config: Similar to above, marked as final location.  
    - Edge cases: Same as above.

  - **First Location**  
    - Type: Limit node  
    - Role: Limits output to first record (used to pick next location).  
    - Edge cases: Empty input.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                          | Input Node(s)                      | Output Node(s)                           | Sticky Note                                                                                                                     |
|-----------------------------|----------------------------------|----------------------------------------|----------------------------------|-----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Telegram Trigger            | Telegram Trigger                 | Entry point for Telegram messages      |                                  | Is Audio?                               | ## 1. Telegram Trigger and message format check                                                                                |
| Is Audio?                  | If                              | Checks if message contains audio       | Telegram Trigger                 | Command (no audio), Collect Audio + Get Locations to Check (audio) | ## 1. Telegram Trigger and message format check                                                                                |
| Command                    | If                              | Checks if message is /start or /help   | Is Audio? (no audio path)        | Command Selection (yes), Wrong Command (no) | ## 1. Telegram Trigger and message format check                                                                                |
| Command Selection          | Switch                          | Routes between /start and /help        | Command                         | Help Message (/help), Collect Remaining Locations (/start) | ## 6. Generate help or error message based on the command selected                                                              |
| Help Message               | Telegram                        | Sends help message                     | Command Selection                |                                         | ## 6. Generate help or error message based on the command selected                                                              |
| Wrong Command              | Telegram                        | Sends wrong command warning            | Command                        |                                         | ## 1. Telegram Trigger and message format check                                                                                 |
| Collect Remaining Locations | Google Sheets                   | Reads unchecked locations              | Command Selection (/start)       | Combine with Location Records           | ## 2. Get all the locations not yet counted                                                                                     |
| Combine with Location Records| Merge                          | Joins location data with sheet records| Collect Remaining Locations      | Last Location ?                         | ## 2. Get all the locations not yet counted                                                                                     |
| Last Location ?            | If                              | Checks if only one location remains    | Combine with Location Records    | Send Instruction: Last Location (yes), First Location (no) | ## 7. Get all the locations not yet counted                                                                                     |
| Send Instruction: Last Location | Telegram                    | Sends final location instruction       | Last Location ?                 |                                         | ## 8. Generate a customised instruction to go the next or last location                                                        |
| First Location             | Limit                           | Selects first location from list       | Last Location ? (no path)        | Send Instruction: Next Location          | ## 8. Generate a customised instruction to go the next or last location                                                        |
| Send Instruction: Next Location | Telegram                    | Sends next location instruction        | First Location                  |                                         | ## 8. Generate a customised instruction to go the next or last location                                                        |
| Get Locations to Check     | Google Sheets                   | Reads unchecked locations              | Is Audio? (audio path)           | Extract Location IDs                    | ## 2. Get all the locations not yet counted                                                                                     |
| Extract Location IDs       | Aggregate                      | Aggregates location IDs list            | Get Locations to Check           | Merge Transcript With Location List     | ## 2. Get all the locations not yet counted                                                                                     |
| Merge Transcript With Location List | Merge                  | Combines extracted data with location list| Transcription Error (no error), Extract Location IDs | Is Location in Scope?                     | ## 5. If the location is in the scope of the cycle count, update the inventory quantity                                          |
| Transcribe Operator Command | OpenAI Audio                   | Transcribes audio to text               | Collect Audio                   | Model: Information Extraction            | ## 3. Transcribe and parse the operator audio command using AI                                                                 |
| Model: Information Extraction | OpenAI Chat                  | Extracts structured data from transcription| Transcribe Operator Command     | Get Location, Qty                        | ## 3. Transcribe and parse the operator audio command using AI                                                                 |
| Structured Output Parser   | LangChain Output Parser         | Parses AI output into structured JSON  | Model: Information Extraction   | Get Location, Qty (ai_outputParser)      | ## 3. Transcribe and parse the operator audio command using AI                                                                 |
| Get Location, Qty          | LangChain Agent                | Produces structured location and quantity| Structured Output Parser        | Transcription Error                      | ## 3. Transcribe and parse the operator audio command using AI                                                                 |
| Transcription Error        | If                             | Checks if transcription fields are null| Get Location, Qty               | Ask to Repeat (error), Merge Transcript With Location List (valid) | ## 4. If the location or the quantity is missing, ask the operator to repeat                                                     |
| Ask to Repeat              | Telegram                       | Requests operator to repeat input      | Transcription Error             |                                         | ## 4. If the location or the quantity is missing, ask the operator to repeat                                                     |
| Is Location in Scope?      | If                             | Checks if location exists in remaining list | Merge Transcript With Location List | Update Inventory Quantity (yes), Error Message: Wrong Location (no) | ## 5. If the location is in the scope of the cycle count, update the inventory quantity                                          |
| Update Inventory Quantity  | Google Sheets                  | Updates sheet with actual counted quantity| Is Location in Scope? (yes)     | Wait                                    | ## 5. If the location is in the scope of the cycle count, update the inventory quantity                                          |
| Wait                      | Wait                           | Placeholder delay node                  | Update Inventory Quantity       |                                         |                                                                                                                                 |
| Error Message: Wrong Location | Telegram                    | Sends error if location invalid or counted | Is Location in Scope? (no)      |                                         | ## 5. If the location is in the scope of the cycle count, update the inventory quantity                                          |
| Remaining Locations        | Aggregate                      | Aggregates count of remaining locations| Combine with Location Records   | Number of Locations                     | ## 7. Get all the locations not yet counted                                                                                     |
| Number of Locations        | Set                            | Sets variable with remaining location count| Remaining Locations             | Combine with Location Records           | ## 7. Get all the locations not yet counted                                                                                     |
| Collect Audio             | Telegram                       | Downloads voice message audio           | Is Audio? (audio path)          | Transcribe Operator Command             | ## 3. Transcribe and parse the operator audio command using AI                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node:**  
   - Type: Telegram Trigger  
   - Parameters: Listen for "message" updates.  
   - Credential: Telegram Bot API Token.

2. **Add "Is Audio?" If node:**  
   - Condition: Check if `message.voice.file_id` exists.  
   - Connect Telegram Trigger to Is Audio?.

3. **Add "Command" If node:**  
   - Condition: Check if text message is "/start" or "/help".  
   - Connect Is Audio? (no audio output) to Command.

4. **Add "Command Selection" Switch node:**  
   - Two outputs: "/start" and "/help" based on exact match of message.text.  
   - Connect Command "true" output to Command Selection.

5. **Add "Help Message" Telegram node:**  
   - Message: Detailed help instructions (Markdown).  
   - Connect Command Selection "/help" output here.

6. **Add "Wrong Command" Telegram node:**  
   - Message: Warn user about unrecognized commands and list valid options.  
   - Connect Command "false" output to Wrong Command node.

7. **Add "Collect Remaining Locations" Google Sheets node:**  
   - Operation: Read rows where `checked` column ≠ "X".  
   - Configure Google Sheets credentials, specify spreadsheet ID and sheet name.  
   - Connect Command Selection "/start" output here.

8. **Add "Combine with Location Records" Merge node:**  
   - Mode: SQL Join (combineBySql) to join with remaining locations.  
   - Connect Collect Remaining Locations output to this.

9. **Add "Last Location ?" If node:**  
   - Condition: Check if remaining location count == 1.  
   - Connect Combine with Location Records output here.

10. **Add "Send Instruction: Last Location" Telegram node:**  
    - Message: Instruct operator about final location to count.  
    - Connect Last Location ? "true" output here.

11. **Add "First Location" Limit node:**  
    - Limit to 1 record to select next location.  
    - Connect Last Location ? "false" output here.

12. **Add "Send Instruction: Next Location" Telegram node:**  
    - Message: Instruct operator to proceed to next location.  
    - Connect First Location output here.

13. **Add "Get Locations to Check" Google Sheets node:**  
    - Operation: Read rows where `checked` = "X" (unchecked locations).  
    - Configure credentials and spreadsheet details.

14. **Add "Extract Location IDs" Aggregate node:**  
    - Aggregate all `location_id` into an array.

15. **Add "Collect Audio" Telegram node:**  
    - Download voice file using `voice.file_id` from message.

16. **Add "Transcribe Operator Command" OpenAI audio node:**  
    - Operation: Transcribe audio binary data to text.  
    - Configure OpenAI credentials.

17. **Add "Model: Information Extraction" OpenAI GPT chat node:**  
    - Model: GPT-4.1-mini.  
    - Input: Transcription text.

18. **Add "Structured Output Parser" LangChain node:**  
    - JSON Schema: `{ "location_id": "string|null", "quantity": "string|null" }`.  
    - Connect to GPT chat output.

19. **Add "Get Location, Qty" LangChain agent node:**  
    - System prompt: Extract location and quantity into JSON only.  
    - Connect Structured Output Parser output here.

20. **Add "Transcription Error" If node:**  
    - Checks if either `location_id` or `quantity` is "null".  
    - Connect Get Location, Qty output here.

21. **Add "Ask to Repeat" Telegram node:**  
    - Message: Request operator to repeat voice message clearly.  
    - Connect Transcription Error "true" output here.

22. **Add "Merge Transcript With Location List" Merge node:**  
    - Mode: combineBySql join between transcript output and location list.  
    - Connect Transcription Error "false" output and Extract Location IDs output here.

23. **Add "Is Location in Scope?" If node:**  
    - Condition: Check if extracted `location_id` is in location_ids array.  
    - Connect Merge Transcript With Location List output here.

24. **Add "Update Inventory Quantity" Google Sheets node:**  
    - Operation: Update row matching `location_id` with `actual_quantity`, set `checked` = "V".  
    - Configure credentials and spreadsheet details.  
    - Connect Is Location in Scope? "true" output here.

25. **Add "Wait" node:**  
    - Default parameters; optional delay.  
    - Connect Update Inventory Quantity output here.

26. **Add "Error Message: Wrong Location" Telegram node:**  
    - Message: Notify operator location invalid or already counted.  
    - Connect Is Location in Scope? "false" output here.

27. **Add "Remaining Locations" Aggregate node:**  
    - Aggregates count of remaining locations from Combine with Location Records output.  
    - Connect Combine with Location Records output here.

28. **Add "Number of Locations" Set node:**  
    - Sets variable `remaining_locations` with count from Remaining Locations.  
    - Connect Remaining Locations output here.

29. **Connect Number of Locations output back to Last Location ? node** to close the loop for processing locations.

30. **Wire all nodes accordingly to reflect the above description** ensuring correct data flow and error routing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                             | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| AI Powered Inventory Cycle Count Telegram Bot: workflow uses OpenAI GPT-4.1-mini for extraction and Google Sheets as WMS.                                              | Sticky Note on node "Sticky Note" at position [0,0]                                             |
| Setup requirements: Google Sheet with columns `location_id`, `system_quantity`, `actual_quantity`, `checked`; Telegram Bot with API token; OpenAI credentials.          | Sticky Note on node "Sticky Note" at position [0,0]                                             |
| Customization tips: modify prompts for other languages, include SOPs in Telegram messages, or connect to custom WMS database instead of Google Sheets.                   | Sticky Note on node "Sticky Note" at position [0,0]                                             |
| Workflow demo and explanation video available at: [YouTube Tutorial](https://www.youtube.com/watch?v=_EOJ3M7APsQ)                                                       | Sticky Note1 near node "Help Message"                                                           |
| Telegram messages use Markdown formatting and avoid attribution footer for clarity.                                                                                    | Observed in Telegram nodes' parameters                                                          |
| Workflow uses LangChain nodes for structured AI output parsing to ensure strict JSON compliance and avoid ambiguous outputs.                                           | Observed in Structured Output Parser and Get Location, Qty nodes                               |

---

**Disclaimer:** The text is derived exclusively from an automated workflow created with n8n, adhering strictly to content policies. No illegal or protected content is processed. All data is lawful and publicly accessible.