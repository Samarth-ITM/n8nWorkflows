Telegram AI Assistant with Rate Limiting and Auto-Reset using Google Sheets

https://n8nworkflows.xyz/workflows/telegram-ai-assistant-with-rate-limiting-and-auto-reset-using-google-sheets-7834


# Telegram AI Assistant with Rate Limiting and Auto-Reset using Google Sheets

---

## 1. Workflow Overview

This workflow implements a **Telegram AI Assistant with per-user rate limiting and automatic reset**, leveraging Google Sheets as a persistent backend for tracking user message counts. It is designed to control usage of an AI chatbot to prevent abuse, manage operational costs, and provide a smooth user experience.

### Target Use Cases
- Customer service or demo chatbots with daily or session-based message limits
- Educational bots enforcing usage quotas
- Any AI assistant deployed publicly needing usage control and cost management

### Logical Blocks

- **1.1 Input Reception and User Identification**  
  Receives Telegram messages, extracts user identifiers, and logs or updates usage counters in Google Sheets.

- **1.2 Message Count Management**  
  Retrieves the current message count for the user, increments it, and updates the Google Sheet accordingly.

- **1.3 Rate Limit Enforcement**  
  Uses a Switch node to route users based on their current message count relative to the allowed limit (default 3). It either blocks, warns, or allows AI response.

- **1.4 AI Processing and Response**  
  If under limit, processes the user message through an AI agent powered by Azure OpenAI (GPT-4.1-2) with memory buffer for context, and sends the AI-generated response back to the user.

- **1.5 User Notification on Limit**  
  If user hits the limit exactly, sends a warning message informing them about rate limiting.

- **1.6 Silent Blocking**  
  If user exceeds the limit, performs no operation (no response sent), effectively blocking further messages until reset.

- **1.7 Automatic Reset System**  
  A scheduled workflow branch periodically resets all message counters in the Google Sheet, enabling users to interact again. The reset interval is configurable (testing with minutes, production with hourly/daily).

- **1.8 Documentation and Guidance**  
  Sticky notes provide clear documentation inside the workflow about setup, configuration, and logic for ease of maintenance and customization.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception and User Identification

**Overview:**  
Receives incoming Telegram messages and logs the user’s chat ID in Google Sheets, creating or updating the user record.

**Nodes Involved:**  
- Telegram (Telegram Trigger)  
- Append or update row in sheet (Google Sheets node)

**Node Details:**

- **Telegram (Telegram Trigger)**  
  - Type: Telegram Trigger (Webhook)  
  - Configuration: Listens to "message" updates from Telegram API  
  - Credentials: Telegram API credentials for the AI Agent bot  
  - Inputs: Incoming Telegram message JSON containing user chat ID and text  
  - Outputs: User message and metadata  
  - Edge cases: Telegram API downtime, webhook misconfiguration, malformed updates

- **Append or update row in sheet (Google Sheets)**  
  - Type: Google Sheets node  
  - Configuration:  
    - Operation: Append or Update row  
    - Sheet: Sheet1 of a Google Spreadsheet (ID: 1hBZwpnCr_PZeiALke7ifgBjaUiPWS8Ml6zbbafQbavY)  
    - Matching column: " ID" (Telegram chat ID) to locate user record  
    - Writes user chat ID to " ID" column  
  - Credentials: Google Sheets OAuth2 account  
  - Inputs: Telegram node output (user chat ID)  
  - Outputs: Existing or newly created user row data  
  - Edge cases: Google API rate limits, permission errors, spreadsheet schema mismatch

---

### 2.2 Message Count Management

**Overview:**  
Retrieves the current message count from the sheet, increments it by one, and updates the sheet accordingly.

**Nodes Involved:**  
- Get row(s) in sheet (Google Sheets)  
- Code (JavaScript code node)  
- Append or update row in sheet1 (Google Sheets)

**Node Details:**

- **Get row(s) in sheet (Google Sheets)**  
  - Type: Google Sheets node  
  - Configuration:  
    - Operation: Get row(s)  
    - Filter: Matches " ID" column with Telegram chat ID from input  
    - Range: A:Z to get all columns (including message counter)  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Output from Append or update row in sheet (user record)  
  - Outputs: User row, including current "Message Counter" value  
  - Edge cases: No matching row found (user new), Google API errors

- **Code (JavaScript)**  
  - Type: Code node (JavaScript)  
  - Role: Processes the retrieved row data to increment the message count  
  - Logic:  
    - Extract current message count from "Message Counter" column (defaults to 0 if missing)  
    - Increment count by 1  
    - Output session_id (user ID), new message_count, and row_number for update  
  - Inputs: Output from Get row(s) in sheet  
  - Outputs: JSON object with updated counters  
  - Edge cases: Parsing errors if "Message Counter" is malformed or missing

- **Append or update row in sheet1 (Google Sheets)**  
  - Type: Google Sheets node  
  - Configuration:  
    - Operation: Append or Update row  
    - Matching column: " ID"  
    - Updates "Message Counter" with incremented count  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Output from Code node  
  - Outputs: Updated user record with new message count  
  - Edge cases: API permission issues, update conflicts

---

### 2.3 Rate Limit Enforcement

**Overview:**  
Routes user flow based on message count: under limit → AI response; at limit → warning message; over limit → silent block.

**Nodes Involved:**  
- Switch (conditional routing)  
- No Operation (NoOp)  
- Limit Message (Telegram)  
- Agent (Langchain AI Agent)

**Node Details:**

- **Switch**  
  - Type: Switch node  
  - Configuration:  
    - Conditions on "Message Counter" value:  
      - Route 1: > 3 (over limit)  
      - Route 2: = 3 (at limit)  
      - Route 3: < 3 (under limit)  
  - Inputs: Output from Append or update row in sheet1 (message count)  
  - Outputs: Three branches for different user states  
  - Edge cases: Non-numeric counters, missing data

- **No Operation (NoOp)**  
  - Type: No Operation node  
  - Role: Silently blocks users exceeding limit by halting processing  
  - Inputs: Route 1 from Switch (Message Counter > 3)  
  - Outputs: None  
  - Edge cases: None (pass-through node)

- **Limit Message (Telegram)**  
  - Type: Telegram node (send message)  
  - Configuration:  
    - Sends text: "Daily limit exceeded 🚫. Try again later ⏳."  
    - Chat ID from Telegram message  
  - Credentials: Telegram API  
  - Inputs: Route 2 from Switch (Message Counter = 3)  
  - Outputs: Confirmation of message sent  
  - Edge cases: Telegram API failures

- **Agent (Langchain AI Agent)**  
  - Type: Langchain Agent node  
  - Configuration:  
    - Uses Telegram message text as input prompt  
    - System message defines AI assistant behavior and response guidelines  
    - Output parser enabled (to format AI responses)  
  - Inputs: Route 3 from Switch (Message Counter < 3)  
  - Outputs: AI-generated response text  
  - Edge cases: AI API failures, malformed input text, rate limits

---

### 2.4 AI Processing and Response

**Overview:**  
Processes user messages under limit through Azure OpenAI GPT-4.1-2 model with session memory, then sends response to user.

**Nodes Involved:**  
- Azure (Azure OpenAI GPT-4.1-2)  
- Memory (Langchain Memory Buffer Window)  
- Agent Answer (Telegram)

**Node Details:**

- **Azure (Langchain LM Chat Azure OpenAI)**  
  - Type: Language Model node (Azure OpenAI GPT-4.1-2)  
  - Configuration:  
    - Model: gpt-4.1-2  
    - No additional options set  
  - Credentials: Azure OpenAI API credentials  
  - Inputs: From Agent node as languageModel input  
  - Outputs: AI completion response  
  - Edge cases: API quota exceeded, network issues

- **Memory (Langchain Memory Buffer Window)**  
  - Type: Memory Buffer for AI context  
  - Configuration:  
    - Session key: Telegram chat ID (custom key)  
    - Maintains conversational context per user session  
  - Inputs: AI dialogue context for Agent node  
  - Outputs: Updated memory context for AI  
  - Edge cases: Memory overflow, session key mismatch

- **Agent Answer (Telegram)**  
  - Type: Telegram node (send message)  
  - Configuration:  
    - Sends AI response text from Agent node output  
    - Chat ID from Telegram message  
    - Attribution disabled for clean response  
  - Credentials: Telegram API  
  - Inputs: Output from Agent node  
  - Outputs: Confirmation of message sent  
  - Edge cases: Telegram API errors, message size limits

---

### 2.5 Automatic Reset System

**Overview:**  
Scheduled branch that resets all user message counters to zero at configurable intervals to allow fresh usage cycles.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) in sheet1 (all user records)  
- Update row in sheet (reset counters)

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger node  
  - Configuration:  
    - Interval: Every 1 minute (for testing)  
    - Can be customized to hourly, daily, weekly via cron expressions  
  - Inputs: None (triggered automatically)  
  - Outputs: Initiates reset process  
  - Edge cases: Node downtime, scheduling conflicts

- **Get row(s) in sheet1 (Google Sheets)**  
  - Type: Google Sheets node  
  - Configuration:  
    - Operation: Get all rows without filter (fetch all user records)  
    - Sheet: Same Sheet1 as main flow  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Output from Schedule Trigger  
  - Outputs: List of all user rows with message counters  
  - Edge cases: Large data set delays, API limits

- **Update row in sheet (Google Sheets)**  
  - Type: Google Sheets node  
  - Configuration:  
    - Operation: Update row  
    - Sets "Message Counter" column to zero for each user  
  - Credentials: Google Sheets OAuth2  
  - Inputs: Output from Get row(s) in sheet1 (iterated per row)  
  - Outputs: Confirmation of update  
  - Edge cases: Race conditions if user sends message while resetting, API limits

---

### 2.6 Documentation and Guidance

**Overview:**  
Sticky notes provide embedded documentation and configuration tips for maintainers and users.

**Nodes Involved:**  
- Sticky Note (Automatic Reset System)  
- Sticky Note2 (AI Agent Rate Limiter Overview)  
- Sticky Note3 (Smart Message Tracking)  
- Sticky Note1 (User-Friendly Responses)

**Node Details:**

- Sticky notes contain setup instructions, usage details, customization tips, and explain the logic flow and user experience design.

---

## 3. Summary Table

| Node Name                   | Node Type                                 | Functional Role                             | Input Node(s)                   | Output Node(s)                  | Sticky Note                                                                                             |
|-----------------------------|-------------------------------------------|---------------------------------------------|--------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------|
| Telegram                    | Telegram Trigger                          | Receives Telegram messages                   |                                | Append or update row in sheet   |                                                                                                        |
| Append or update row in sheet| Google Sheets                            | Creates/updates user record with chat ID    | Telegram                       | Get row(s) in sheet             |                                                                                                        |
| Get row(s) in sheet         | Google Sheets                             | Retrieves user row with message count       | Append or update row in sheet   | Code                           |                                                                                                        |
| Code                        | Code (JavaScript)                         | Increments message counter                   | Get row(s) in sheet            | Append or update row in sheet1  |                                                                                                        |
| Append or update row in sheet1| Google Sheets                          | Updates message count for user               | Code                          | Switch                        |                                                                                                        |
| Switch                      | Switch                                   | Routes based on message count limit          | Append or update row in sheet1 | No Operation, Limit Message, Agent | Sticky Note3 covers logic here: explains routing for limits (<3, =3, >3)                               |
| No Operation                | No Operation                             | Silently blocks users over limit             | Switch (over limit)            |                                | Sticky Note1 explains silent blocking behavior                                                        |
| Limit Message               | Telegram                                 | Sends limit warning message                   | Switch (at limit)              |                                | Sticky Note1 explains limit warning message                                                           |
| Agent                       | Langchain Agent                          | Processes user input and prepares AI prompt | Switch (under limit)           | Agent Answer                   | Sticky Note1 explains normal AI response                                                             |
| Azure                       | Langchain LM Chat Azure OpenAI           | Generates AI completion (GPT-4.1-2)          | Agent                         | Agent                         |                                                                                                        |
| Memory                      | Langchain Memory Buffer Window            | Maintains session memory context             | Agent                         | Agent                         |                                                                                                        |
| Agent Answer                | Telegram                                 | Sends AI response to user                     | Agent                         |                                |                                                                                                        |
| Schedule Trigger            | Schedule Trigger                         | Periodically triggers reset process           |                                | Get row(s) in sheet1           | Sticky Note explains automatic reset system and configuration options                                 |
| Get row(s) in sheet1        | Google Sheets                            | Retrieves all user records for reset          | Schedule Trigger              | Update row in sheet            |                                                                                                        |
| Update row in sheet         | Google Sheets                            | Resets message counters to zero               | Get row(s) in sheet1           |                                |                                                                                                        |
| Sticky Note                 | Sticky Note                             | Documentation of reset system                  |                                |                                | Covers automatic reset system details                                                                 |
| Sticky Note2                | Sticky Note                             | Overall workflow description and usage tips   |                                |                                | Covers AI Agent Rate Limiter overview                                                                 |
| Sticky Note3                | Sticky Note                             | Explains message tracking and switch logic   |                                |                                | Covers smart message tracking and switch logic                                                       |
| Sticky Note1                | Sticky Note                             | Describes user response types                 |                                |                                | Covers user-friendly responses including blocking and warnings                                       |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Configure webhook with your Telegram bot token credentials  
   - Set updates to listen for "message" events  
   - Position at start of workflow  

2. **Create Google Sheets Node to Append or Update User Record**  
   - Type: Google Sheets  
   - Operation: Append or Update row  
   - Spreadsheet ID: Your Google Sheets document ID  
   - Sheet Name: Sheet1 (or your chosen sheet)  
   - Matching columns: " ID" (Telegram chat ID)  
   - Map " ID" column to expression: `{{$json["message"]["chat"]["id"]}}`  
   - Connect Telegram Trigger output to this node  
   - Add Google Sheets OAuth2 credentials  

3. **Create Google Sheets Node to Get User Row**  
   - Type: Google Sheets  
   - Operation: Get row(s)  
   - Filter: " ID" equals `{{$json["message"]["chat"]["id"]}}`  
   - Range: A:Z to get full row  
   - Connect output of previous Google Sheets node  
   - Use same credentials  

4. **Create Code Node to Increment Counter**  
   - Type: Code (JavaScript)  
   - Paste code logic:  
     ```js
     const currentRow = $input.first().json;
     const currentCount = currentRow['Message Counter'] || 0;
     const newCount = parseInt(currentCount) + 1;
     return { json: { session_id: currentRow.A, message_count: newCount, row_number: currentRow.__rowNum } };
     ```  
   - Connect output of "Get row(s) in sheet" node  
   
5. **Create Google Sheets Node to Update Message Counter**  
   - Type: Google Sheets  
   - Operation: Append or Update row  
   - Matching columns: " ID"  
   - Map " ID" to `{{$json["session_id"]}}`  
   - Map "Message Counter" to `{{$json["message_count"]}}`  
   - Connect output of Code node  
   - Use same credentials  

6. **Create Switch Node to Enforce Rate Limit**  
   - Type: Switch  
   - Input property: `{{$json["Message Counter"]}}`  
   - Conditions:  
     - Route 1: Greater than 3 (over limit)  
     - Route 2: Equals 3 (at limit)  
     - Route 3: Less than 3 (under limit)  
   - Connect output of Google Sheets update node  

7. **Create No Operation Node for Over Limit**  
   - Type: No Operation  
   - Connect Route 1 output of Switch node  

8. **Create Telegram Node to Send Limit Warning**  
   - Type: Telegram  
   - Text: "Daily limit exceeded 🚫. Try again later ⏳."  
   - Chat ID: `{{$json["message"]["chat"]["id"]}}`  
   - Connect Route 2 output of Switch node  
   - Add Telegram API credentials  

9. **Create Langchain Agent Node for AI Processing**  
   - Type: Langchain Agent  
   - Text input: `{{$node["Telegram"].json["message"]["text"]}}` or equivalent  
   - System message: Detailed AI assistant instructions (see overview)  
   - Enable output parser  
   - Connect Route 3 output of Switch node  

10. **Create Azure OpenAI Node**  
    - Type: Langchain LM Chat Azure OpenAI  
    - Model: gpt-4.1-2  
    - Connect AI languageModel input from Agent node  
    - Add Azure OpenAI credentials  

11. **Create Memory Buffer Node**  
    - Type: Langchain Memory Buffer Window  
    - Session key: `{{$json["message"]["chat"]["id"]}}`  
    - Connect AI memory input to Agent node  

12. **Connect Agent output to Azure and Memory nodes**  
    - Azure output connects back to Agent node’s languageModel input  
    - Memory output connects back to Agent node’s memory input  

13. **Create Telegram Node to Send AI Response**  
    - Type: Telegram  
    - Text: `{{$json["output"]}}` (from Agent node output)  
    - Chat ID: `{{$json["message"]["chat"]["id"]}}`  
    - Connect output of Agent node  
    - Use Telegram credentials  

14. **Create Schedule Trigger Node for Reset**  
    - Type: Schedule Trigger  
    - Set interval (e.g., every 1 minute for testing)  
    - No input  

15. **Create Google Sheets Node to Get All User Rows**  
    - Type: Google Sheets  
    - Operation: Get all rows (no filter)  
    - Connect output of Schedule Trigger  
    - Use same credentials  

16. **Create Google Sheets Node to Update Rows and Reset Counters**  
    - Type: Google Sheets  
    - Operation: Update row  
    - For each row, set "Message Counter" to "0"  
    - Connect output of Get all rows node (enable batch processing or looping)  
    - Use same credentials  

17. **Add Sticky Notes** (optional but recommended)  
    - Add nodes of type Sticky Note with contents:  
      - Overall workflow description and tips  
      - Explanation of message tracking and Switch logic  
      - User response types  
      - Automatic reset system configuration and advice  

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                        | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Workflow solves the problem of controlling AI chatbot usage per user to prevent abuse and manage costs effectively.                                               | Workflow description                                                                                           |
| Google Sheets columns required: "ID" (Telegram chat.id) in Column A, "Message Counter" in Column B                                                                | Google Sheets setup instructions                                                                               |
| The Switch node conditions can be customized to change the message limit per user                                                                                 | Switch node configuration                                                                                      |
| Automatic reset intervals can be configured using cron expressions for production: hourly, daily, weekly                                                         | Sticky Note on reset system                                                                                     |
| AI assistant system message includes guidance on tone, response style, limitations awareness, and user safety prioritization                                     | Agent node system message                                                                                       |
| Telegram nodes use credentials linked to the AI bot; ensure proper permissions and webhook setup                                                                  | Telegram API credentials                                                                                        |
| Azure OpenAI API usage requires GPT-4.1-2 model credentials; monitor usage quotas                                                                                  | Azure OpenAI credentials                                                                                        |
| Memory buffer maintains conversational context per user session to enable coherent AI dialogue                                                                   | Memory node configuration                                                                                       |
| Rate limit warning message and silent blocking help maintain user experience and control spam or abuse                                                           | Sticky Note1 content                                                                                            |
| Testing uses short reset intervals (minutes); production should use longer intervals to balance cost and user experience                                         | Sticky Note on reset system                                                                                     |
| For high-traffic scenarios, consider optimizing Google Sheets API usage or switching to a more scalable database solution                                        | General scalability note                                                                                        |

---

**Disclaimer:** The provided content originates exclusively from an n8n automated workflow. It strictly adheres to content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.

---