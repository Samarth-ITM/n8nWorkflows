Log Food Calories from Images to Google Sheets using LINE and OpenAI Vision

https://n8nworkflows.xyz/workflows/log-food-calories-from-images-to-google-sheets-using-line-and-openai-vision-7228


# Log Food Calories from Images to Google Sheets using LINE and OpenAI Vision

### 1. Workflow Overview

This workflow enables a LINE chatbot to receive food-related inputs (text or images) from users, analyze these inputs with OpenAI’s models, estimate calorie content for foods detected in images, and log the results in Google Sheets. It then sends real-time feedback messages back to the user on LINE.

**Target Use Cases:**  
- Logging calorie information of meals from user-shared photos via LINE.  
- Allowing users to query or interact via text messages through LINE, answered by an AI assistant.  
- Maintaining a persistent calorie log in Google Sheets for personal tracking or analysis.

**Logical Blocks:**  
- **1.1 Input Reception & User Verification:** Receives webhook events from LINE, verifies authorized users, and routes depending on message type (text or image).  
- **1.2 Text Message Handling:** Processes text messages by forwarding them to an AI agent who generates a response sent back to LINE user.  
- **1.3 Image Message Handling:** Downloads images from LINE, analyzes images with OpenAI Vision for food calorie estimation, formats and extracts structured data from AI responses, composes user-friendly messages, logs data to Google Sheets, and sends confirmation messages back to the user.  
- **1.4 Data Parsing & Formatting:** Includes JavaScript nodes that parse JSON or fallback extract calories and dish names from AI responses, and prepare data for LINE messages and Google Sheets.  
- **1.5 Output & Feedback:** Sends processed results or error messages back to the LINE user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & User Verification

**Overview:**  
Handles incoming requests from LINE’s webhook, checks if the user is authorized, and routes the message based on its type (text or image).

**Nodes Involved:**  
- LINE webhook  
- user verification (If node)  
- Switch  

**Node Details:**  
- **LINE webhook**  
  - Type: Webhook  
  - Role: Entry point for POST requests from LINE Messaging API  
  - Config: Path set to a unique webhook ID, listens to POST  
  - Input: HTTP POST from LINE  
  - Output: JSON event data passed downstream  
  - Edge cases: Unauthorized requests, invalid webhook format  
- **user verification**  
  - Type: If  
  - Role: Filters requests by matching the LINE userId to a predefined authorized userId  
  - Config: Compares `{{$json.body.events[0].source.userId}}` with a string placeholder `{your id}`  
  - Input: JSON from webhook  
  - Output: Passes only authorized users forward  
  - Edge cases: Unauthorized user messages will stop workflow or be ignored  
- **Switch**  
  - Type: Switch  
  - Role: Routes messages based on message type field (`text` vs `image`)  
  - Config: Checks `{{$json.body.events[0].message.type}}` for "text" or "image"  
  - Output connections:  
    - "text" → "only message" node  
    - "image" → "images download" node  
    - fallback → ignored or extra processing (not used here)  
  - Edge cases: Unknown message types are not processed.

---

#### 1.2 Text Message Handling

**Overview:**  
Extracts the text content from the webhook, sends it to an AI agent (powered by LangChain and OpenAI Chat), then sends the AI-generated reply back to the LINE user.

**Nodes Involved:**  
- only message (Set node)  
- AI Agent  
- OpenAI Chat Model (used as AI language model in AI Agent)  
- send LINE  

**Node Details:**  
- **only message**  
  - Type: Set  
  - Role: Extracts the user’s text message from webhook JSON to a simplified variable `text`  
  - Config: Sets `text = {{$json.body.events[0].message.text}}`  
  - Input: JSON event  
  - Output: JSON with `text` property  
- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Processes input text with AI assistant logic, uses OpenAI Chat Model for responses  
  - Config: Input text from `{{$json.text}}`, system message sets assistant role  
  - Connections: Linked to OpenAI Chat Model as language model and Simple Memory for context  
  - Output: AI response text (in `output` property)  
  - Edge cases: OpenAI API errors, rate limits, empty input  
- **OpenAI Chat Model**  
  - Type: LangChain Chat Model (OpenAI GPT-4.1-mini)  
  - Role: Provides the AI language model service for the AI Agent  
  - Config: Model set to GPT-4.1-mini, credentials linked to OpenAI API  
- **send LINE**  
  - Type: HTTP Request  
  - Role: Sends reply message back to LINE Messaging API using the replyToken  
  - Config: POST to LINE’s reply endpoint with JSON body containing the AI-generated message from AI Agent’s output  
  - Headers: Authorization Bearer token (LINE channel access token), Content-Type application/json  
  - Edge cases: HTTP failures, invalid token, LINE API errors

---

#### 1.3 Image Message Handling

**Overview:**  
Downloads image content sent by the user, analyzes it with OpenAI Vision API to estimate calories for food items, parses AI output, prepares messages and data entries, logs results in Google Sheets, and sends confirmation message back to LINE user.

**Nodes Involved:**  
- images download  
- Analyze image  
- Code (parse AI JSON and normalize)  
- Code1 (format message for LINE)  
- Edit Fields1 (set output message)  
- Append row in sheet  
- send LINE  

**Node Details:**  
- **images download**  
  - Type: HTTP Request  
  - Role: Downloads image content from LINE API using message ID  
  - Config: URL uses message ID from webhook, includes Authorization header with LINE channel access token  
  - Input: JSON with message ID  
  - Output: Binary image data (base64)  
  - Edge cases: Invalid token, expired message ID, network errors  
- **Analyze image**  
  - Type: LangChain OpenAI Image Analysis node  
  - Role: Sends image base64 to OpenAI Vision model with prompt to estimate calories only if image contains food, returns structured JSON  
  - Config: Model chatgpt-4o-latest, resource = image, operation = analyze, inputType = base64  
  - Output: AI JSON response with dishName and calories array  
  - Edge cases: No food detected, model errors, malformed response  
- **Code**  
  - Type: Code (JavaScript)  
  - Role: Parses AI response JSON, handles potential formatting/escaping issues, extracts dish names and calories, normalizes data, handles errors gracefully, attempts manual extraction if JSON invalid  
  - Input: AI response JSON from Analyze image  
  - Output: Array of normalized JSON objects with dishName, calories, timestamps  
  - Edge cases: JSON parse errors, missing fields, unexpected response formats  
- **Code1**  
  - Type: Code (JavaScript)  
  - Role: Converts parsed food calorie data into a user-friendly LINE message string, aggregates calories, handles no food detected scenario, outputs message JSON for sending  
  - Input: Output from Code node  
  - Output: JSON containing formatted message text and metadata  
  - Edge cases: Empty data, malformed input  
- **Edit Fields1**  
  - Type: Set  
  - Role: Simplifies and sets the message output for the send LINE node  
  - Config: Assigns `output = {{$json.message}}`  
- **Append row in sheet**  
  - Type: Google Sheets  
  - Role: Appends one row per dish with columns: date (today’s date), menu (dishName), cal (calories)  
  - Config: Document and sheet IDs set to specific Google Sheet, columns mapped explicitly  
  - Credentials: Google Sheets OAuth2  
  - Edge cases: API quota, invalid sheet ID, data type mismatches  
- **send LINE**  
  - Same as in text handling, sends the confirmation message to the user

---

#### 1.4 Data Parsing & Formatting (Code Nodes)

**Overview:**  
These custom JavaScript nodes clean and parse AI responses, extract structured data, handle errors, and generate user-friendly messages.

**Nodes involved:**  
- Code (parsing AI JSON for calories)  
- Code1 (formatting LINE message)

**Node Details:**  
- Both nodes use robust string manipulation and regex patterns to extract JSON arrays from AI responses, handle markdown code blocks, and fallback to manual extraction if needed.  
- They ensure calories are numeric, dish names exist, and timestamps are added.  
- They log processing steps and errors to console for debugging.  
- They handle cases where AI returns no valid data or malformed JSON gracefully.

---

#### 1.5 Output & Feedback

**Overview:**  
Final step that sends the AI-generated or processed message back to the LINE user to confirm results or inform of errors.

**Nodes involved:**  
- send LINE (shared by both text and image handling paths)  

**Node Details:**  
- Uses LINE replyToken from webhook to send messages  
- Supports sending text messages only  
- Includes error handling for API issues

---

### 3. Summary Table

| Node Name         | Node Type                            | Functional Role                         | Input Node(s)           | Output Node(s)         | Sticky Note                        |
|-------------------|------------------------------------|---------------------------------------|------------------------|-----------------------|----------------------------------|
| LINE webhook      | Webhook                            | Entry point for LINE webhook events   | —                      | user verification     | See Sticky Note                   |
| user verification | If                                 | Verify authorized user by userId      | LINE webhook           | Switch                | See Sticky Note                   |
| Switch            | Switch                            | Route by message type (text/image)    | user verification      | only message, images download | See Sticky Note                   |
| only message      | Set                               | Extract text message from webhook     | Switch                 | AI Agent              | See Sticky Note                   |
| AI Agent          | LangChain Agent                   | Process text message with AI assistant | only message, Simple Memory, OpenAI Chat Model | send LINE       | See Sticky Note                   |
| OpenAI Chat Model | LangChain Chat Model (OpenAI)     | Provide GPT-4.1-mini AI model          | AI Agent (ai_languageModel) | AI Agent              |                                  |
| Simple Memory     | LangChain Memory Buffer           | Manage session context for AI agent   | —                      | AI Agent (ai_memory)   |                                  |
| images download   | HTTP Request                     | Download image content from LINE API  | Switch                 | Analyze image         | See Sticky Note                   |
| Analyze image     | LangChain OpenAI Image Analysis  | Analyze image to estimate calories    | images download        | Code, Code1           | See Sticky Note                   |
| Code              | Code (JavaScript)                  | Parse and normalize AI JSON calorie data | Analyze image          | Append row in sheet    | See Sticky Note2                  |
| Code1             | Code (JavaScript)                  | Format calorie data into LINE message | Analyze image          | Edit Fields1           | See Sticky Note1                  |
| Edit Fields1      | Set                               | Prepare message output for LINE       | Code1                   | send LINE              | See Sticky Note1                  |
| Append row in sheet | Google Sheets                    | Append calorie log row to Google Sheet | Code                    | —                      | See Sticky Note2                  |
| send LINE         | HTTP Request                     | Send reply message back to LINE user  | AI Agent, Edit Fields1  | —                      | See Sticky Note                   |
| Sticky Note       | Sticky Note                      | Documentation and instructions        | —                      | —                      | Contains detailed workflow notes |
| Sticky Note1      | Sticky Note                      | Message to LINE                        | —                      | —                      | "## message to LINE"             |
| Sticky Note2      | Sticky Note                      | Write to Google Sheets                 | —                      | —                      | "## write to Google Sheets"      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create LINE webhook node:**  
   - Type: Webhook (POST)  
   - Path: Unique webhook path (e.g., "0980b700-284e-4e52-a33e-71e087eea37f")  
   - Purpose: Receive incoming LINE events.  

2. **Add user verification node:**  
   - Type: If  
   - Condition: `{{$json.body.events[0].source.userId}} == "{your id}"`  
   - Purpose: Filter messages from authorized users only.  

3. **Add a Switch node:**  
   - Type: Switch  
   - Condition: `{{$json.body.events[0].message.type}}`  
   - Routes:  
       - "text" → "only message" node  
       - "image" → "images download" node  

4. **Set node "only message":**  
   - Type: Set  
   - Assign variable `text = {{$json.body.events[0].message.text}}`  

5. **Add LangChain AI Agent node:**  
   - Type: LangChain Agent  
   - Input: `{{$json.text}}`  
   - System message: "You are an AI assistant. Your job is to respond to user messages."  
   - Use OpenAI Chat Model as language model (GPT-4.1-mini)  
   - Attach Simple Memory node for session context keyed by `{{$json.body.events[0].source.userId}}`  

6. **Create OpenAI Chat Model node:**  
   - Type: LangChain LM Chat OpenAI  
   - Model: GPT-4.1-mini (or suitable GPT-4 variant)  
   - Credentials: OpenAI API key  

7. **Add HTTP Request node to send reply on LINE:**  
   - URL: `https://api.line.me/v2/bot/message/reply`  
   - Method: POST  
   - Headers: Authorization Bearer `{channel access token}`, Content-Type: application/json  
   - Body (JSON): Use replyToken from webhook and AI Agent output message  

8. **Create "images download" node:**  
   - Type: HTTP Request  
   - URL: `https://api-data.line.me/v2/bot/message/{{$json.body.events[0].message.id}}/content`  
   - Headers: Authorization Bearer `{channel access token}`  
   - Output: Binary image data (base64)  

9. **Add "Analyze image" node:**  
   - Type: LangChain OpenAI (Image Analysis)  
   - Model: chatgpt-4o-latest or similar OpenAI Vision model  
   - Input: base64 image from previous node  
   - Prompt: Request calorie estimation only if food detected, output JSON array with dishName and calories  

10. **Add JavaScript "Code" node (parse AI JSON):**  
    - Parse AI response JSON (handling markdown code blocks if present)  
    - Normalize objects with dishName and calories fields  
    - Convert calories to integer, add timestamps  
    - Attempt manual extraction on parse errors  
    - Output array of structured data  

11. **Add JavaScript "Code1" node (format LINE message):**  
    - Take parsed data from Code node  
    - Generate user-friendly multiline text listing dishes and calories  
    - Handle empty or error cases with fallback messages  

12. **Add "Edit Fields1" Set node:**  
    - Set field `output = {{$json.message}}` for sending message  

13. **Add Google Sheets node:**  
    - Operation: Append row  
    - Document ID and Sheet ID: Your Google Sheets document and sheet for logging  
    - Columns: date (current date), menu (dishName), cal (calories)  
    - Credentials: Google Sheets OAuth2  

14. **Connect "Code" node output to Google Sheets node input.**

15. **Connect "Code1" output to "Edit Fields1" then to "send LINE" node.**

16. **Connect "AI Agent" output to "send LINE" node for text messages path.**

17. **Set all credentials:**  
    - LINE Messaging API channel access token for HTTP Request nodes  
    - OpenAI API key for LangChain nodes  
    - Google Sheets OAuth2 for Google Sheets node  

18. **Test by sending text and image messages to your LINE bot.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                    | Context or Link                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow integrates LINE Messaging API with OpenAI Vision for food calorie estimation and Google Sheets for logging. Replace placeholders with your own LINE userId, channel access token, and Google Sheets IDs.                             | Sticky Note content in workflow                                                                              |
| OpenAI Vision model "chatgpt-4o-latest" is used for image analysis and calorie estimation; ensure your OpenAI plan supports it.                                                                                                               | OpenAI documentation                                                                                         |
| Use Google Sheets with OAuth2 credentials configured in n8n for appending rows securely.                                                                                                                                                       | Google Sheets API docs                                                                                        |
| To enable session context via Simple Memory, userId from LINE webhook is used as session key for AI conversations.                                                                                                                             | LangChain and n8n LangChain integration docs                                                                 |
| LINE reply messages use replyToken from original webhook event; ensure token is valid and used only once per event.                                                                                                                             | LINE Messaging API reply message guidelines                                                                  |
| This workflow includes robust error handling and manual fallback extraction for AI JSON parsing, improving resilience against unexpected AI output formats.                                                                                    | Custom JavaScript code nodes                                                                                  |
| Video demo and setup instructions available on public n8n community forums and blogs (search for "LINE food calorie logger n8n").                                                                                                            | n8n community and blog resources                                                                              |
| Remember to activate the workflow and set it to listen for incoming LINE webhook events to function live.                                                                                                                                     | n8n workflow activation                                                                                        |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, respecting all content policies and handling only legal and public data.