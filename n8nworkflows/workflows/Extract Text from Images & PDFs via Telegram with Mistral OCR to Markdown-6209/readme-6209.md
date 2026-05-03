Extract Text from Images & PDFs via Telegram with Mistral OCR to Markdown

https://n8nworkflows.xyz/workflows/extract-text-from-images---pdfs-via-telegram-with-mistral-ocr-to-markdown-6209


# Extract Text from Images & PDFs via Telegram with Mistral OCR to Markdown

---

## 1. Workflow Overview

This workflow provides a complete solution for performing Optical Character Recognition (OCR) on image and PDF files sent via Telegram. Users interact with a Telegram bot by sending supported files (PNG, JPEG, PDF), which are then processed using Mistral OCR to extract text. The extracted text is formatted into Markdown and returned to the user as a downloadable `.md` file. The workflow also includes features for file size validation, optional user access whitelist, Telegram webhook management, and secure file delivery via a proxy method.

### Logical Blocks:

- **1.1 Incoming Request Handling**  
  Receives and identifies the type of incoming Telegram updates, including messages and callback queries.

- **1.2 Whitelist Access Control**  
  Optionally restricts bot use to authorized Telegram user IDs.

- **1.3 File Validation and Classification**  
  Checks file size limits and determines if the file is a supported type (image or PDF).

- **1.4 File Download Link Generation**  
  Uses Telegram API to generate a temporary file link for OCR processing.

- **1.5 OCR Processing with Mistral**  
  Sends the file link to Mistral OCR API and retrieves extracted text.

- **1.6 Markdown Conversion and Delivery**  
  Converts OCR results into Markdown format, creates a text file, and sends it back to the user via Telegram.

- **1.7 Telegram Bot Webhook Setup and Management**  
  Provides manual triggers and nodes to configure Telegram webhook URLs for development and production.

- **1.8 Error Handling and User Notifications**  
  Manages different error scenarios like invalid files, oversized files, access denial, and OCR failures, informing users accordingly.

---

## 2. Block-by-Block Analysis

### 2.1 Incoming Request Handling

**Overview:**  
Handles all incoming Telegram updates via webhook, distinguishing between message updates and callback queries, and routes accordingly for further processing.

**Nodes Involved:**  
- Incoming request (Webhook)  
- Settings (Set)  
- Determine Incoming Request Source Type (Switch)  
- Telegram Event Handler (Switch)  
- Is Whitelist Disabled? (If)  
- Whitelist Logic (Code)  
- Check Whitelist Status (If)  
- Access Denied (Telegram)  
- Status “Typing…” (Telegram)  
- Determine Message Type (Switch)  
- Command Router (Switch)  
- Notification about correct commands (Telegram)  
- Inform Bot Capabilities (Telegram)  
- Command Router outputs

**Node Details:**

- **Incoming request**  
  - Type: Webhook  
  - Role: Entry point to receive Telegram updates (POST/GET) with binary and JSON data  
  - Configuration: Path set for webhook, supports POST and GET, uses responseNode mode  
  - Inputs: External Telegram webhook calls  
  - Outputs: Sends data to Settings node  
  - Edge cases: Incorrect webhook calls, malformed data

- **Settings**  
  - Type: Set  
  - Role: Sets runtime parameters such as whitelist toggle, allowed chat IDs, production URL, and current chat ID  
  - Key parameters:  
    - Bot_Whitelist_Active (boolean)  
    - Allowed_Chat_IDs (string, comma-separated)  
    - File_Downloader_Prod_URL (string)  
    - Chat_ID (dynamic from incoming message or callback query)  
  - Outputs: Forwarded to Determine Incoming Request Source Type  
  - Edge cases: Missing or malformed parameters

- **Determine Incoming Request Source Type**  
  - Type: Switch  
  - Role: Identifies if incoming request is a Telegram message (POST) or a file proxy request (GET with tg_file query)  
  - Outputs: Routes to Telegram Event Handler or Get a file node  
  - Edge cases: Missing expected properties, unrecognized request type

- **Telegram Event Handler**  
  - Type: Switch  
  - Role: Differentiates between normal messages and callback queries for further processing  
  - Outputs: Routes to whitelist check or denial paths  
  - Edge cases: Unexpected Telegram update types

- **Is Whitelist Disabled?**  
  - Type: If  
  - Role: Checks if whitelist feature is disabled, routes accordingly  
  - Outputs: To Status “Typing…” or Whitelist Logic  
  - Edge cases: Improper boolean evaluation

- **Whitelist Logic**  
  - Type: Code  
  - Role: Checks if the current chat ID is in the allowed list using regex matching  
  - Logic: Splits Allowed_Chat_IDs string, escapes regex characters, tests Chat_ID  
  - Outputs: isWhitelisted boolean  
  - Edge cases: Empty whitelist, incorrect formatting in Allowed_Chat_IDs

- **Check Whitelist Status**  
  - Type: If  
  - Role: Acts on whitelist check result; allows or denies access  
  - Outputs: Status “Typing…” for allowed or Access Denied for denied users  
  - Edge cases: False positives/negatives in whitelist logic

- **Access Denied**  
  - Type: Telegram node (send message)  
  - Role: Notifies user that access is denied due to whitelist restrictions  
  - Configuration: Sends Markdown-formatted message  
  - Edge cases: Telegram API failures

- **Status “Typing…”**  
  - Type: Telegram node (sendChatAction)  
  - Role: Sends "typing" status to Telegram chat as user interaction feedback  
  - Edge cases: Telegram rate limits or API errors

- **Determine Message Type**  
  - Type: Switch  
  - Role: Determines if incoming message is a bot command, a file document, or a callback, routing accordingly  
  - Outputs: Routes to Command Router, file size checking, or notifications  
  - Edge cases: Missing or malformed message entities

- **Command Router**  
  - Type: Switch  
  - Role: Routes commands such as "/start" to appropriate handlers  
  - Outputs: Triggers Inform Bot Capabilities or default path  
  - Edge cases: Unknown commands

- **Inform Bot Capabilities**  
  - Type: Telegram node (send message)  
  - Role: Sends introductory message about bot capabilities when "/start" command is received  
  - Edge cases: Telegram API errors

- **Notification about correct commands**  
  - Type: Telegram node (send message)  
  - Role: Informs users when message is not a file or command (i.e., no processing occurs)  
  - Edge cases: Telegram API errors

---

### 2.2 Whitelist Access Control

**Overview:**  
Implements an optional whitelist feature that restricts bot interaction to specified Telegram User IDs.

**Nodes Involved:**  
- Settings  
- Whitelist Logic (Code)  
- Check Whitelist Status (If)  
- Access Denied (Telegram)  
- Is Whitelist Disabled? (If)  
- Status “Typing…” (Telegram)

**Node Details:**  
Covered above in Block 2.1; the whitelist logic node performs regex matching on chat IDs, and access is denied or granted accordingly.

Edge cases include empty whitelist strings, malformed user ID lists, or disabled whitelist toggle.

---

### 2.3 File Validation and Classification

**Overview:**  
Validates the file size against Telegram Bot API limits and classifies the file type to ensure it is supported (PNG, JPEG, or PDF).

**Nodes Involved:**  
- Checking file size (If)  
- Maximum file size exceeded (Telegram)  
- File classifier (Code)  
- Image / PDF (If)  
- Invalid file (Telegram)

**Node Details:**

- **Checking file size**  
  - Type: If  
  - Role: Verifies that the uploaded document's file size is ≤ 20 MB (20,971,520 bytes)  
  - Output: Passes to File classifier if valid, otherwise to Maximum file size exceeded node  
  - Edge cases: Missing file_size property, files slightly over limit

- **Maximum file size exceeded**  
  - Type: Telegram node (send message)  
  - Role: Notifies user that their file exceeds the 20 MB limit  
  - Edge cases: Telegram API failures

- **File classifier**  
  - Type: Code  
  - Role: Determines if file is an image (JPEG/PNG) or PDF document based on MIME type  
  - Logic: Sets `file_class` to "image_url", "document_url", or "false" if unsupported  
  - Outputs: Routes to Image / PDF node  
  - Edge cases: Unexpected MIME types, missing MIME type field

- **Image / PDF**  
  - Type: If  
  - Role: Checks if `file_class` is not "false" to confirm supported file type  
  - Outputs: Generates temporary file link or triggers invalid file notification  
  - Edge cases: File classified incorrectly

- **Invalid file**  
  - Type: Telegram node (send message)  
  - Role: Informs user that only PDF, JPG, and PNG files are accepted  
  - Edge cases: Telegram API failures

---

### 2.4 File Download Link Generation

**Overview:**  
Uses Telegram API to generate a temporary download link for the user-uploaded file, which will be passed to Mistral OCR.

**Nodes Involved:**  
- Generating temporary file link (Telegram)  
- Mistral OCR (HTTP Request)  
- Sticky Note10 (Informational)  

**Node Details:**

- **Generating temporary file link**  
  - Type: Telegram API node (get file link)  
  - Role: Generates a temporary URL for Mistral OCR to fetch the file from Telegram servers  
  - Input: File ID from incoming message  
  - Output: Passes file URL to Mistral OCR  
  - Edge cases: Telegram API limits, expired links

- **Mistral OCR**  
  - Type: HTTP Request  
  - Role: Sends the temporary file URL to Mistral OCR API for text extraction  
  - Configuration:  
    - POST request to https://api.mistral.ai/v1/ocr  
    - JSON body includes model "mistral-ocr-latest" and document type and URL  
    - Uses predefined Mistral Cloud API credentials  
  - Error handling: On error, continues with an error output  
  - Edge cases: API timeouts, invalid credentials, malformed request body

- **Sticky Note10**  
  - Purpose: Documents that Mistral downloads the file through the temporary link generated via Telegram, requiring the workflow to be active for this proxy to work

---

### 2.5 OCR Processing with Mistral

**Overview:**  
Processes the OCR response from Mistral, converting the extracted text for further use.

**Nodes Involved:**  
- Mistral OCR (continued)  
- Markdown converter (Code)  
- Problem with file recognition (Telegram)

**Node Details:**

- **Markdown converter**  
  - Type: Code  
  - Role: Converts the OCR response pages into Markdown format  
  - Logic:  
    - Validates presence of `pages` array  
    - For each page, prepends a heading "Page X" and appends page markdown content  
    - Joins pages with separator `\n\n---\n\n`  
    - Returns JSON with `fileName` and combined `fileContent`  
  - Edge cases: Missing or malformed OCR data, exceptions thrown if input is invalid

- **Problem with file recognition**  
  - Type: Telegram node (send message)  
  - Role: Informs the user if OCR processing failed  
  - Edge cases: Telegram API errors

---

### 2.6 Markdown Conversion and Delivery

**Overview:**  
Converts the Markdown text into a file and sends it as a downloadable document back to the user on Telegram.

**Nodes Involved:**  
- Converting Markdown to File (ConvertToFile)  
- Send Markdown File to Telegram (Telegram)

**Node Details:**

- **Converting Markdown to File**  
  - Type: ConvertToFile  
  - Role: Converts Markdown string (`fileContent`) to a UTF-8 encoded text file  
  - Output: Binary file data prepared for Telegram sendDocument operation

- **Send Markdown File to Telegram**  
  - Type: Telegram node (sendDocument)  
  - Role: Sends the converted Markdown file back to the user's chat  
  - File name is dynamically set to original file name with suffix `-ocr.md`  
  - Edge cases: Telegram API failures, file size issues

---

### 2.7 Telegram Bot Webhook Setup and Management

**Overview:**  
Supports manual configuration and switching between development and production webhook URLs for the Telegram bot.

**Nodes Involved:**  
- Manual Webhook Setup Trigger (Manual Trigger)  
- Telegram Webhook Configuration (Set)  
- Check Production Mode (If)  
- Set Production Webhook (HTTP Request)  
- Set Development Webhook (HTTP Request)  
- Return Webhook Status (HTTP Request)  
- Sticky Note2 (Instructional)

**Node Details:**

- **Manual Webhook Setup Trigger**  
  - Type: Manual trigger  
  - Role: Initiates webhook configuration flow manually

- **Telegram Webhook Configuration**  
  - Type: Set  
  - Role: Stores configuration parameters such as bot API key, webhook URLs, and mode flag

- **Check Production Mode**  
  - Type: If  
  - Role: Checks if production mode is active to choose webhook URL

- **Set Production Webhook**  
  - Type: HTTP Request  
  - Role: Calls Telegram API to set production webhook URL

- **Set Development Webhook**  
  - Type: HTTP Request  
  - Role: Calls Telegram API to set development webhook URL, drops pending updates

- **Return Webhook Status**  
  - Type: HTTP Request  
  - Role: Retrieves current webhook info from Telegram API

- **Sticky Note2**  
  - Provides detailed manual instructions for setting up the Telegram bot webhook using BotFather and copying URLs

---

### 2.8 Error Handling and User Notifications

**Overview:**  
Handles notifying users about errors such as invalid files, oversized files, unauthorized access, and OCR failures.

**Nodes Involved:**  
- Invalid file (Telegram)  
- Maximum file size exceeded (Telegram)  
- Access Denied (Telegram)  
- Problem with file recognition (Telegram)  
- Notification about correct commands (Telegram)

**Details:**  
These nodes send user-friendly messages formatted in Markdown to inform about the specific errors encountered.

---

## 3. Summary Table

| Node Name                      | Node Type               | Functional Role                                 | Input Node(s)                    | Output Node(s)                         | Sticky Note                                                                                                    |
|--------------------------------|-------------------------|------------------------------------------------|---------------------------------|--------------------------------------|---------------------------------------------------------------------------------------------------------------|
| Incoming request                | Webhook                 | Entry point for Telegram updates                | External Telegram webhook calls | Settings                             |                                                                                                               |
| Settings                       | Set                     | Define runtime parameters                        | Incoming request                | Determine Incoming Request Source Type | Sticky Note3 (Settings Configuration details)                                                                 |
| Determine Incoming Request Source Type | Switch                  | Distinguish POST message or file proxy request  | Settings                       | Telegram Event Handler, Get a file   |                                                                                                               |
| Telegram Event Handler          | Switch                  | Distinguish messages vs callback queries        | Determine Incoming Request Source Type | Is Whitelist Disabled?           |                                                                                                               |
| Is Whitelist Disabled?          | If                      | Check if whitelist is active                     | Telegram Event Handler          | Status “Typing…”, Whitelist Logic    | Sticky Note9 (Whitelist block explanation)                                                                    |
| Whitelist Logic                | Code                    | Check if user is in whitelist                    | Is Whitelist Disabled?          | Check Whitelist Status               |                                                                                                               |
| Check Whitelist Status          | If                      | Grant or deny access                             | Whitelist Logic                | Status “Typing…”, Access Denied      |                                                                                                               |
| Access Denied                  | Telegram                 | Notify user of access denial                      | Check Whitelist Status          | None                               |                                                                                                               |
| Status “Typing…”               | Telegram                 | Send typing status to user                        | Check Whitelist Status, Is Whitelist Disabled? | Determine Message Type             |                                                                                                               |
| Determine Message Type          | Switch                  | Identify message type (command, file, callback) | Status “Typing…”                | Command Router, Checking file size, Notification about correct commands |                                                                                                               |
| Command Router                 | Switch                  | Route commands like /start                        | Determine Message Type          | Inform Bot Capabilities             |                                                                                                               |
| Inform Bot Capabilities         | Telegram                 | Send bot introduction message                     | Command Router                 | None                               |                                                                                                               |
| Notification about correct commands | Telegram                 | Notify user for unsupported messages             | Determine Message Type          | None                               |                                                                                                               |
| Checking file size             | If                      | Validate file size ≤ 20 MB                        | Determine Message Type          | File classifier, Maximum file size exceeded |                                                                                                               |
| Maximum file size exceeded      | Telegram                 | Notify user file is too large                     | Checking file size              | None                               |                                                                                                               |
| File classifier               | Code                    | Determine if file is image or PDF                 | Checking file size              | Image / PDF                        |                                                                                                               |
| Image / PDF                   | If                      | Check valid file class                            | File classifier                | Generating temporary file link, Invalid file |                                                                                                               |
| Invalid file                   | Telegram                 | Notify user of invalid file type                  | Image / PDF                    | None                               |                                                                                                               |
| Generating temporary file link | Telegram                 | Get temporary download URL for file               | Image / PDF                    | Mistral OCR                       | Sticky Note10 (File download access explanation)                                                              |
| Mistral OCR                   | HTTP Request            | Send file URL to Mistral OCR API                  | Generating temporary file link | Markdown converter, Problem with file recognition |                                                                                                               |
| Markdown converter            | Code                    | Convert OCR pages to Markdown                      | Mistral OCR                   | Converting Markdown to File         |                                                                                                               |
| Problem with file recognition  | Telegram                 | Notify user of OCR failure                         | Mistral OCR                   | None                               |                                                                                                               |
| Converting Markdown to File    | ConvertToFile           | Convert Markdown string to text file               | Markdown converter            | Send Markdown File to Telegram      |                                                                                                               |
| Send Markdown File to Telegram | Telegram                 | Send Markdown file back to user                    | Converting Markdown to File    | None                               |                                                                                                               |
| Get a file                    | Telegram                 | Download file for proxy requests                    | Determine Incoming Request Source Type | Respond with attachment          |                                                                                                               |
| Respond with attachment        | RespondToWebhook        | Respond with binary file data                       | Get a file                    | None                               |                                                                                                               |
| Manual Webhook Setup Trigger   | Manual Trigger          | Trigger webhook setup flow manually                 | None                          | Telegram Webhook Configuration      |                                                                                                               |
| Telegram Webhook Configuration | Set                     | Set webhook configuration parameters               | Manual Webhook Setup Trigger   | Check Production Mode              | Sticky Note2 (Telegram Bot Webhook Setup instructions)                                                        |
| Check Production Mode          | If                      | Choose between production or development webhook   | Telegram Webhook Configuration | Set Production Webhook, Set Development Webhook |                                                                                                               |
| Set Production Webhook         | HTTP Request            | Set the Telegram webhook URL for production         | Check Production Mode          | Return Webhook Status              |                                                                                                               |
| Set Development Webhook        | HTTP Request            | Set the Telegram webhook URL for development        | Check Production Mode          | Return Webhook Status              |                                                                                                               |
| Return Webhook Status          | HTTP Request            | Get current webhook status from Telegram API        | Set Production Webhook, Set Development Webhook | None                               |                                                                                                               |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Node: "Incoming request"**  
   - Type: Webhook  
   - HTTP Methods: POST, GET  
   - Path: set a unique webhook path  
   - Response Mode: responseNode  
   - Enable binary data property: `data`

2. **Add a "Set" Node: "Settings"**  
   - Define parameters:  
     - `Bot_Whitelist_Active` (boolean, default false)  
     - `Allowed_Chat_IDs` (string, comma-separated IDs)  
     - `File_Downloader_Prod_URL` (string, production URL of Incoming request)  
     - `Chat_ID` (expression: `{{$json.body?.message?.chat?.id || $json.body?.callback_query?.from?.id}}`)  
   - Connect from "Incoming request"

3. **Add a "Switch" Node: "Determine Incoming Request Source Type"**  
   - Output 1 "POST": Condition - `{{$json.body.message.message_id || ""}}` is not empty  
   - Output 2 "File Proxy": Condition - `{{$json.query.tg_file || ""}}` is not empty  
   - Connect from "Settings"

4. **Add a "Switch" Node: "Telegram Event Handler"**  
   - Output 1 "Message": Condition - `{{$json.body.hasOwnProperty('message')}}` true  
   - Output 2 "Callback": Condition - `{{$json.body.hasOwnProperty('callback_query')}}` true  
   - Connect from "Determine Incoming Request Source Type" POST output

5. **Add an "If" Node: "Is Whitelist Disabled?"**  
   - Condition: `{{$node["Settings"].json.Bot_Whitelist_Active}}` is false  
   - Connect from "Telegram Event Handler" outputs

6. **Add a "Code" Node: "Whitelist Logic"**  
   - JavaScript code to parse Allowed_Chat_IDs, check if Chat_ID is in list via regex matching  
   - Connect from "Is Whitelist Disabled?" false output

7. **Add an "If" Node: "Check Whitelist Status"**  
   - Condition: `{{$node["Whitelist Logic"].json.isWhitelisted}}` is true  
   - Connect from "Whitelist Logic"

8. **Add a Telegram node: "Access Denied"** to notify unauthorized users  
   - Message: "Oops! You don't have access to this bot." (Markdown)  
   - Connect from "Check Whitelist Status" false output

9. **Add a Telegram node: "Status “Typing…”"**  
   - Operation: sendChatAction with action "typing"  
   - Connect from "Check Whitelist Status" true output and "Is Whitelist Disabled?" true output

10. **Add a "Switch" Node: "Determine Message Type"**  
    - Output "Bot Command": first message entity type equals "bot_command"  
    - Output "File": message.document exists  
    - Output "Callback": callback_query.data exists  
    - Connect from "Status “Typing…”"

11. **Add a "Switch" Node: "Command Router"**  
    - Output "Start": message text contains "/start"  
    - Connect from "Determine Message Type" "Bot Command" output

12. **Add a Telegram node: "Inform Bot Capabilities"**  
    - Send introductory text about the bot and file size limit  
    - Connect from "Command Router" "Start" output

13. **Add an "If" Node: "Checking file size"**  
    - Condition: `message.document.file_size` ≤ 20971520 bytes (20 MB)  
    - Connect from "Determine Message Type" "File" output

14. **Add a Telegram node: "Maximum file size exceeded"**  
    - Notify user that file size is too large  
    - Connect from "Checking file size" false output

15. **Add a "Code" Node: "File classifier"**  
    - JavaScript to check MIME type of uploaded file: set `file_class` to "image_url" for JPEG/PNG, "document_url" for PDF, or "false" for others  
    - Connect from "Checking file size" true output

16. **Add an "If" Node: "Image / PDF"**  
    - Condition: `file_class` not equals "false"  
    - Connect from "File classifier"

17. **Add a Telegram node: "Invalid file"**  
    - Notify user that only PDF, JPG, PNG files are accepted  
    - Connect from "Image / PDF" false output

18. **Add a Telegram node: "Generating temporary file link"**  
    - Operation: get file info from Telegram using `message.document.file_id`  
    - Connect from "Image / PDF" true output

19. **Add an HTTP Request node: "Mistral OCR"**  
    - POST to `https://api.mistral.ai/v1/ocr`  
    - Body JSON:  
      ```json
      {
        "model": "mistral-ocr-latest",
        "document": {
          "type": "{{ $json.file_class }}",
          "{{ $json.file_class }}": "{{ $json.File_Downloader_Prod_URL.trim() }}?tg_file={{ $json.result.file_id }}"
        }
      }
      ```  
    - Use Mistral Cloud API credentials  
    - Connect from "Generating temporary file link"

20. **Add a "Code" Node: "Markdown converter"**  
    - Validate `pages` array and convert each page's markdown to a combined Markdown string with page headings and separators  
    - Connect from "Mistral OCR" successful output

21. **Add a Telegram node: "Problem with file recognition"**  
    - Notify user if OCR failed  
    - Connect from "Mistral OCR" error output

22. **Add a "ConvertToFile" Node: "Converting Markdown to File"**  
    - Convert fileContent to UTF-8 text file  
    - Connect from "Markdown converter"

23. **Add a Telegram node: "Send Markdown File to Telegram"**  
    - Send document operation with binary data, file name suffix `-ocr.md`  
    - Connect from "Converting Markdown to File"

24. **Add Telegram node: "Get a file"**  
    - Operation: get file by file ID from query parameter `tg_file` (for proxy file download)  
    - Connect from "Determine Incoming Request Source Type" File Proxy output

25. **Add a Respond To Webhook node: "Respond with attachment"**  
    - Responds with binary file data from "Get a file"  
    - Connect from "Get a file"

26. **Add Manual Trigger node: "Manual Webhook Setup Trigger"**  
    - Connects to "Telegram Webhook Configuration"

27. **Add Set node: "Telegram Webhook Configuration"**  
    - Parameters: Production_Mode (boolean), Bot_API_Key (string), Webhook_Prod_URL (string), Webhook_Dev_URL (string)  
    - Connect from Manual Webhook Setup Trigger

28. **Add If node: "Check Production Mode"**  
    - Condition true if Production_Mode is true  
    - Connect from Telegram Webhook Configuration

29. **Add HTTP Request nodes: "Set Production Webhook" and "Set Development Webhook"**  
    - Calls Telegram API for setting webhook to respective URLs, include `drop_pending_updates=1` for dev  
    - Connect from Check Production Mode outputs

30. **Add HTTP Request node: "Return Webhook Status"**  
    - Calls Telegram API to get webhook info  
    - Connect from both Set Production Webhook and Set Development Webhook

31. **Add sticky notes for documentation** at appropriate locations, including Settings configuration, Webhook setup instructions, Whitelist explanations, and file download proxy notes.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| This n8n template provides a complete solution for **Optical Character Recognition (OCR)** of image and PDF files directly within Telegram. Key features include effortless OCR via Telegram, file size validation (20 MB limit), Mistral-powered recognition, Markdown output, secure file delivery via a proxy link, optional whitelist security, and simplified webhook management.                                                                                                                                                                                                                      | Sticky Note1 (Workflow description)   |
| The `Settings` node centralizes bot configuration including whitelist activation and allowed user IDs, as well as URLs necessary for file download by Mistral. It is critical to correctly set `File_Downloader_Prod_URL` with a publicly accessible URL that includes port number if non-standard.                                                                                                                                                                                                                                                                                                                     | Sticky Note3 (Settings configuration) |
| Whitelist feature is optional but recommended for security. It restricts bot interactions to specified Telegram user IDs, enhancing control over who can use the OCR functionality.                                                                                                                                                                                                                                                                                                                                                                                                                           | Sticky Note9 (Whitelist explanation)  |
| Telegram file downloader block enables Mistral to fetch files uploaded by users securely via a temporary link generated by Telegram, requiring the workflow to be active for this proxying to function correctly.                                                                                                                                                                                                                                                                                                                                                                                             | Sticky Note10 (File download access)  |
| Webhook setup instructions provide detailed guidance for creating a Telegram bot via BotFather, setting bot API keys, and configuring webhook URLs for development and production environments, facilitating smooth deployment and testing.                                                                                                                                                                                                                                                                                                                                                                   | Sticky Note2 (Webhook setup guide)    |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, a workflow automation tool. The processing strictly complies with valid content policies and contains no illegal or protected elements. All data handled is legal and public.

---