Get Daily Exercise Plan with Flex Message via LINE

https://n8nworkflows.xyz/workflows/get-daily-exercise-plan-with-flex-message-via-line-2988


# Get Daily Exercise Plan with Flex Message via LINE

### 1. Workflow Overview

This workflow, titled **Get Daily Exercise Plan with Flex Message via LINE (YogiAI)**, automates the delivery of daily yoga pose reminders to users through LINE Push Messages. It integrates Google Sheets as a data source for yoga poses, Azure OpenAI for dynamic content generation and formatting, and LINE Messaging API for rich, interactive message delivery.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow daily at 21:30 Bangkok time.
- **1.2 Data Retrieval**: Fetches yoga pose data (names, image URLs, and links) from Google Sheets.
- **1.3 Content Generation**: Uses Azure OpenAI to create user-friendly text messages and formats them for LINE.
- **1.4 JSON Flex Message Construction**: Generates and validates JSON structures for LINE Flex Messages to display yoga poses with images and links.
- **1.5 Message Delivery**: Sends the combined text and Flex Message carousel to users via LINE Push API.
- **1.6 Interaction Logging**: Records sent messages and pose details back into Google Sheets for tracking and improving recommendations.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow automatically every day at 21:30 (9:30 PM) Bangkok time to ensure timely delivery of yoga pose reminders.

- **Nodes Involved:**  
  - Trigger 2130 YogaPosesToday

- **Node Details:**  
  - **Trigger 2130 YogaPosesToday**  
    - Type: Schedule Trigger  
    - Configuration: Set to trigger daily at 21:30 hours (Bangkok timezone)  
    - Inputs: None (time-based trigger)  
    - Outputs: Initiates the workflow chain by triggering the next node "Get PoseName"  
    - Edge Cases: Missed triggers if n8n server is down or timezone misconfiguration  

---

#### 1.2 Data Retrieval

- **Overview:**  
  Retrieves yoga pose data from a specified range in a Google Sheet, including pose names, image URLs, and clickable links.

- **Nodes Involved:**  
  - Get PoseName  
  - Code

- **Node Details:**  
  - **Get PoseName**  
    - Type: Google Sheets  
    - Configuration: Reads range B18:D28 from the "NotePad" sheet in the specified Google Sheet document  
    - Credentials: Google Sheets OAuth2  
    - Inputs: Trigger node output  
    - Outputs: Raw pose data rows with PoseName, uri (image URL), and url (link)  
    - Edge Cases: Google Sheets API errors, empty or malformed data, permission issues  

  - **Code**  
    - Type: Code (JavaScript)  
    - Configuration: Processes all rows from "Get PoseName" to:  
      - Format a text block listing each pose with its image and URL  
      - Extract a list of pose names only for further AI processing  
    - Inputs: Output from "Get PoseName"  
    - Outputs: JSON with two fields:  
      - `outputText`: formatted text with pose details  
      - `poseNamesOnly`: newline-separated list of pose names  
    - Edge Cases: Unexpected data structure, missing fields  

---

#### 1.3 Content Generation

- **Overview:**  
  Uses Azure OpenAI to generate engaging, user-friendly text messages with emojis and clear instructions about the yoga poses for the day. Then reformats the text specifically for LINE messaging.

- **Nodes Involved:**  
  - WritePosesToday  
  - RewritePosesToday

- **Node Details:**  
  - **WritePosesToday**  
    - Type: Langchain Chain LLM (Azure OpenAI)  
    - Configuration:  
      - Prompt instructs the AI to act as an experienced yoga instructor, encouraging focus on today's poses and including related variations.  
      - Input text includes the pose names list from the "Code" node.  
      - Temperature set to 0.8 for creative output.  
    - Inputs: Output from "WriteJSONflex" (via connection)  
    - Outputs: AI-generated text message with friendly tone and emojis  
    - Edge Cases: AI service downtime, rate limits, unexpected output format  

  - **RewritePosesToday**  
    - Type: Langchain Chain LLM (Azure OpenAI)  
    - Configuration:  
      - Reformats the AI-generated text to be chat-friendly for LINE, adding emojis before pose names.  
      - Splits message into up to 3 parts if too long, separated by "======".  
      - Temperature set to 0.9 for varied output.  
    - Inputs: Output text from "WritePosesToday"  
    - Outputs: Formatted text ready for LINE messaging  
    - Edge Cases: Text splitting errors, formatting issues  

---

#### 1.4 JSON Flex Message Construction

- **Overview:**  
  Constructs the JSON payload for LINE Flex Messages, enabling a carousel display of yoga pose images with clickable links. Validates and fixes JSON formatting to comply with LINE API requirements.

- **Nodes Involved:**  
  - WriteJSONflex  
  - Structured Output Parser1  
  - Auto-fixing Output Parser1  
  - CombineAll  
  - Fix JSON

- **Node Details:**  
  - **WriteJSONflex**  
    - Type: Langchain Chain LLM (Azure OpenAI)  
    - Configuration:  
      - Generates JSON bubbles for each yoga pose with image URLs and links, formatted per LINE Flex Message specs.  
      - Uses AI to parse and output JSON for all poses from "Get PoseName".  
      - Has output parser enabled for structured output.  
    - Inputs: Output from "Code" node (formatted text)  
    - Outputs: JSON array of Flex Message bubbles  
    - Edge Cases: JSON syntax errors, incomplete data  

  - **Structured Output Parser1**  
    - Type: Langchain Structured Output Parser  
    - Configuration: Validates JSON schema for Flex Message bubbles as per LINE documentation  
    - Inputs: Output from "WriteJSONflex"  
    - Outputs: Parsed and validated JSON  
    - Edge Cases: Schema mismatch, parsing errors  

  - **Auto-fixing Output Parser1**  
    - Type: Langchain Auto-fixing Output Parser  
    - Configuration: Automatically corrects JSON formatting errors  
    - Inputs: Output from "Structured Output Parser1"  
    - Outputs: Corrected JSON  
    - Edge Cases: Unfixable JSON errors  

  - **CombineAll**  
    - Type: Set Node  
    - Configuration:  
      - Combines the rewritten text message and Flex Message JSON bubbles into a single JSON payload for LINE Push API.  
      - Replaces newlines and removes markdown/tags for clean text.  
      - Sets recipient user ID ("to") for LINE message delivery.  
    - Inputs: Output from "RewritePosesToday" and "WriteJSONflex"  
    - Outputs: Complete LINE message JSON body  
    - Edge Cases: Incorrect user ID, malformed JSON  

  - **Fix JSON**  
    - Type: Langchain Chain LLM (Azure OpenAI)  
    - Configuration:  
      - Final JSON validation and fixing step before sending to LINE API.  
      - Ensures JSON is properly formatted and returns only the fixed JSON.  
    - Inputs: Output from "CombineAll"  
    - Outputs: Valid JSON for LINE API  
    - Edge Cases: JSON errors, AI service issues  

---

#### 1.5 Message Delivery

- **Overview:**  
  Sends the final combined text and Flex Message carousel to the user via LINE Push Message API.

- **Nodes Involved:**  
  - Line Push with Flex Bubble

- **Node Details:**  
  - **Line Push with Flex Bubble**  
    - Type: HTTP Request  
    - Configuration:  
      - POST request to `https://api.line.me/v2/bot/message/push`  
      - Sends JSON body from "Fix JSON" node  
      - Uses HTTP Header Authentication with LINE channel access token  
      - The "to" field in JSON must be replaced with the recipient's LINE user ID  
    - Inputs: Valid JSON from "Fix JSON"  
    - Outputs: API response from LINE server  
    - Edge Cases: Authentication failure, network errors, invalid user ID, rate limits  

---

#### 1.6 Interaction Logging

- **Overview:**  
  Logs the sent message details and individual yoga pose sequences back into Google Sheets for tracking user engagement and refining pose selection.

- **Nodes Involved:**  
  - YogaLog  
  - AI Agent  
  - Split Out  
  - YogaLog2

- **Node Details:**  
  - **YogaLog**  
    - Type: Google Sheets  
    - Configuration: Appends a new row to the "YogaLog" sheet with:  
      - Date (timestamp from trigger)  
      - JSON payload sent to LINE  
      - Text message content  
    - Inputs: Output from "Line Push with Flex Bubble" (via "AI Agent")  
    - Outputs: Confirmation of append operation  
    - Edge Cases: Google Sheets API errors, permission issues  

  - **AI Agent**  
    - Type: Langchain Agent (Azure OpenAI)  
    - Configuration:  
      - Processes text to ensure pose names match Google Sheets data and formats JSON accordingly (without emojis).  
      - Uses structured output parser to extract pose sequences and names.  
    - Inputs: Output from "YogaLog"  
    - Outputs: Structured JSON with pose sequences and names  
    - Edge Cases: AI parsing errors, mismatched data  

  - **Split Out**  
    - Type: Split Out  
    - Configuration: Splits the array of yoga poses from AI Agent output into individual items for logging  
    - Inputs: Output from "AI Agent"  
    - Outputs: Individual pose items for logging  
    - Edge Cases: Empty arrays, malformed data  

  - **YogaLog2**  
    - Type: Google Sheets  
    - Configuration: Appends each pose item with Date, Pose name, and Sequence number into "YogaLog2" sheet  
    - Inputs: Output from "Split Out"  
    - Outputs: Confirmation of append operation  
    - Edge Cases: Google Sheets API errors, data mismatch  

---

### 3. Summary Table

| Node Name                  | Node Type                                   | Functional Role                          | Input Node(s)                      | Output Node(s)                   | Sticky Note                                                                                                  |
|----------------------------|---------------------------------------------|----------------------------------------|----------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Trigger 2130 YogaPosesToday | Schedule Trigger                            | Daily scheduled trigger at 21:30       | None                             | Get PoseName                    |                                                                                                              |
| Get PoseName               | Google Sheets                              | Retrieve yoga pose data from sheet     | Trigger 2130 YogaPosesToday      | Code                           | Get the Data: fetches PoseName, uri, url from Google Sheets; update with your own data                        |
| Code                       | Code (JavaScript)                          | Format pose data text and extract names| Get PoseName                    | WriteJSONflex                  |                                                                                                              |
| WriteJSONflex              | Langchain Chain LLM (Azure OpenAI)         | Generate JSON Flex Message bubbles     | Code                           | WritePosesToday                | Write FlexMessage for Images: JSON for LINE Flex Messages; use auto-parser; see LINE docs and simulator       |
| WritePosesToday            | Langchain Chain LLM (Azure OpenAI)         | Generate user-friendly text with emojis| WriteJSONflex                  | RewritePosesToday              | Write Text for Poses today: AI rewrites text with emojis for user-friendly messages                           |
| RewritePosesToday          | Langchain Chain LLM (Azure OpenAI)         | Format text specifically for LINE      | WritePosesToday                | CombineAll                    |                                                                                                              |
| CombineAll                 | Set Node                                  | Combine text and Flex JSON for LINE    | RewritePosesToday, WriteJSONflex| Fix JSON                      | Combine the result and push via LINE; replace "to" with your UID; create authorization header                |
| Fix JSON                   | Langchain Chain LLM (Azure OpenAI)         | Validate and fix JSON before sending   | CombineAll                    | Line Push with Flex Bubble     |                                                                                                              |
| Line Push with Flex Bubble | HTTP Request                              | Send message to LINE Push API           | Fix JSON                      | YogaLog                      |                                                                                                              |
| YogaLog                    | Google Sheets                              | Log sent message JSON and text          | Line Push with Flex Bubble      | AI Agent                     | Write back the data into Log and Log2; track sent poses for weighted randomness                               |
| AI Agent                   | Langchain Agent (Azure OpenAI)              | Parse and validate pose data for logging| YogaLog                      | Split Out                    |                                                                                                              |
| Split Out                  | Split Out                                 | Split pose array into individual items | AI Agent                      | YogaLog2                     |                                                                                                              |
| YogaLog2                   | Google Sheets                              | Log individual pose sequences           | Split Out                    | None                         |                                                                                                              |
| Sticky Note                | Sticky Note                               | Workflow description                    | None                         | None                         | YogiAI: daily reminders via LINE Push; data from Google Sheets weighted random poses                         |
| Sticky Note1               | Sticky Note                               | Data retrieval explanation              | None                         | None                         | Get the Data: Google Sheets range and sample data link                                                       |
| Sticky Note2               | Sticky Note                               | Flex Message JSON explanation           | None                         | None                         | Write FlexMessage for Images; LINE Flex Message docs and simulator links                                     |
| Sticky Note3               | Sticky Note                               | Text generation explanation             | None                         | None                         | Write Text for Poses today: AI rewriting with emojis                                                        |
| Sticky Note4               | Sticky Note                               | Combining and sending message explanation| None                         | None                         | Combine result and push via LINE; instructions for "to" field and authorization                             |
| Sticky Note5               | Sticky Note                               | Logging explanation                      | None                         | None                         | Write back data into Log and Log2; weighted randomness explained                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Scheduled Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 21:30 (Bangkok timezone)  

2. **Add Google Sheets Node "Get PoseName"**  
   - Type: Google Sheets  
   - Configure to read range B18:D28 from the "NotePad" sheet of your Google Sheet document  
   - Use Google Sheets OAuth2 credentials  
   - Connect output from Schedule Trigger  

3. **Add Code Node "Code"**  
   - Type: Code (JavaScript)  
   - Paste the following code to format pose data and extract pose names:  
     ```javascript
     const items = $input.all();

     let outputText = "";
     let poseNamesList = [];

     items.forEach(item => {
       const { PoseName, uri, url } = item.json;
       outputText += `Name: ${PoseName}\nuri: ${uri}\nurl: ${url}\n\n`;
       poseNamesList.push(PoseName);
     });

     return [
       {
         json: {
           outputText,
           poseNamesOnly: poseNamesList.join('\n')
         }
       }
     ];
     ```  
   - Connect output from "Get PoseName"  

4. **Add Langchain Chain LLM Node "WriteJSONflex"**  
   - Type: Langchain Chain LLM (Azure OpenAI)  
   - Set model to "4o" with temperature 0.9  
   - Prompt: Instruct AI to generate JSON bubbles for each pose with image URL and clickable link, following LINE Flex Message format (see LINE docs)  
   - Enable output parser with JSON schema validation  
   - Connect output from "Code"  

5. **Add Langchain Chain LLM Node "WritePosesToday"**  
   - Type: Langchain Chain LLM (Azure OpenAI)  
   - Model: "4o", temperature 0.8  
   - Prompt: Act as yoga instructor, encourage focus on today's poses, include all poses from list with related variations and emojis  
   - Input text: Use `poseNamesOnly` from "Code" node  
   - Connect output from "WriteJSONflex"  

6. **Add Langchain Chain LLM Node "RewritePosesToday"**  
   - Type: Langchain Chain LLM (Azure OpenAI)  
   - Model: "4o", temperature 0.9  
   - Prompt: Format text with emojis before pose names, split into up to 3 messages if too long  
   - Connect output from "WritePosesToday"  

7. **Add Set Node "CombineAll"**  
   - Type: Set  
   - Create a field `LineBody` with JSON combining:  
     - `to`: your LINE user ID (replace with actual UID)  
     - `messages`: array with:  
       - Text message (from `RewritePosesToday`), sanitized for newlines and markdown  
       - Flex message carousel with contents from all JSON bubbles generated by "WriteJSONflex"  
   - Connect output from "RewritePosesToday" and "WriteJSONflex"  

8. **Add Langchain Chain LLM Node "Fix JSON"**  
   - Type: Langchain Chain LLM (Azure OpenAI)  
   - Model: "4o"  
   - Prompt: Fix and return only the corrected JSON from `LineBody`  
   - Connect output from "CombineAll"  

9. **Add HTTP Request Node "Line Push with Flex Bubble"**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.line.me/v2/bot/message/push`  
   - Authentication: HTTP Header Auth with your LINE channel access token  
   - Body: JSON from "Fix JSON" node  
   - Connect output from "Fix JSON"  

10. **Add Google Sheets Node "YogaLog"**  
    - Type: Google Sheets  
    - Operation: Append  
    - Sheet: "YogaLog" in your Google Sheet document  
    - Columns:  
      - Date: timestamp from trigger node  
      - JSON: JSON message body sent to LINE  
      - Text: formatted text message sent  
    - Connect output from "Line Push with Flex Bubble"  

11. **Add Langchain Agent Node "AI Agent"**  
    - Type: Langchain Agent (Azure OpenAI)  
    - Prompt: Ensure pose names match Google Sheets data, format JSON without emojis for logging  
    - Connect output from "YogaLog"  

12. **Add Split Out Node "Split Out"**  
    - Type: Split Out  
    - Field to split: `output.yogaPoses` from AI Agent output  
    - Connect output from "AI Agent"  

13. **Add Google Sheets Node "YogaLog2"**  
    - Type: Google Sheets  
    - Operation: Append  
    - Sheet: "YogaLog2" in your Google Sheet document  
    - Columns:  
      - Date: timestamp from trigger  
      - Pose: pose name from split item  
      - Sequence: pose sequence number  
    - Connect output from "Split Out"  

14. **Add Sticky Notes** (optional for documentation)  
    - Add notes describing each block and instructions for data setup, LINE channel setup, and AI usage  

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| YogiAI provides daily yoga pose reminders via LINE Push Messages using Google Sheets data and Azure OpenAI.    | Workflow purpose and overview                                                                       |
| Google Sheets sample data and structure: PoseName, uri (image URL), url (clickable link).                       | https://docs.google.com/spreadsheets/d/1eqLJsUL_QkOMy_qPzNCrUCZdx36asC8P1i3PowTQqLY/edit?usp=sharing |
| LINE Flex Message documentation and simulator for JSON formatting and testing.                                 | https://developers.line.biz/en/docs/messaging-api/using-flex-messages/                              |
| LINE Messaging API for sending push messages; replace "to" field with your LINE user ID and set authorization. | https://developers.line.biz/en/docs/messaging-api/sending-messages/                                |
| Azure OpenAI account required for AI text generation and JSON formatting.                                      | Azure OpenAI portal and API setup                                                                   |
| Weighted random pose selection and logging to improve future recommendations.                                  | Explained in Sticky Note5                                                                           |

---

This document fully describes the YogiAI workflow, enabling advanced users and AI agents to understand, reproduce, and maintain the automation with clear insights into each functional block, node configuration, and integration points.