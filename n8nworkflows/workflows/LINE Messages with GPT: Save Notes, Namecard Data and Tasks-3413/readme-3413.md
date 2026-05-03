LINE Messages with GPT: Save Notes, Namecard Data and Tasks

https://n8nworkflows.xyz/workflows/line-messages-with-gpt--save-notes--namecard-data-and-tasks-3413


# LINE Messages with GPT: Save Notes, Namecard Data and Tasks

### 1. Workflow Overview

This workflow, titled **"LINE Messages with GPT: Save Notes, Namecard Data and Tasks"**, is designed to automate the processing of incoming messages from the LINE messaging platform. It targets professionals, teams, developers, and business owners who want to streamline message handling by integrating LINE with Microsoft Teams, Microsoft To Do, OneDrive, and OpenRouter.ai.

The workflow logically divides into the following blocks:

- **1.1 Input Reception and User Feedback**  
  Receives incoming LINE messages via webhook and immediately sends a loading animation to the user to indicate processing.

- **1.2 Message Type Routing**  
  Uses a Switch node to categorize incoming messages into tasks, plain text, images, audio, or unsupported types, routing each to appropriate processing paths.

- **1.3 Text Message Processing**  
  Handles plain text messages by saving them into Microsoft Teams channels and replying to the user.

- **1.4 Task Creation from Text Starting with "T "**  
  Extracts tasks from text messages that start with "T ", creates tasks in Microsoft To Do, and sends confirmation replies.

- **1.5 Image Processing and Identification**  
  Downloads images from LINE, then uses OpenRouter.ai via a LangChain agent to classify the image as a namecard, handwritten/text note, or other.

- **1.6 Namecard Data Extraction and Task Creation**  
  For images identified as namecards, extracts structured contact data using OpenRouter.ai, creates follow-up tasks in Microsoft To Do, and sends replies.

- **1.7 Other Image Content Processing**  
  For images that are handwritten notes or other types, extracts text or descriptions, uploads images to OneDrive, posts messages with images to Microsoft Teams, and replies to the user.

- **1.8 Unsupported Message Handling**  
  For unsupported types (e.g., audio), sends polite replies indicating the limitation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and User Feedback

- **Overview:**  
  This block receives incoming messages from LINE via a webhook and immediately sends a loading animation to inform users that their message is being processed.

- **Nodes Involved:**  
  - Line Webhook  
  - Line Loading Animation  
  - Sticky Note1 (documentation)  
  - Sticky Note2 (documentation)

- **Node Details:**  
  - **Line Webhook**  
    - Type: Webhook (HTTP POST)  
    - Config: Listens on path `/minibear` for POST requests from LINE platform  
    - Input: Incoming LINE message event JSON  
    - Output: Passes event data downstream  
    - Potential Failures: Misconfigured webhook URL, invalid LINE signature, network issues  
    - Sticky Note1 explains setup instructions and LINE webhook configuration link.

  - **Line Loading Animation**  
    - Type: HTTP Request  
    - Config: POST to LINE API endpoint `/v2/bot/chat/loading/start` with JSON body containing `chatId` (userId from webhook event) and loading duration (60 seconds)  
    - Authentication: HTTP Header Auth with LINE token  
    - Input: Receives webhook event data  
    - Output: Passes data to Switch node  
    - Potential Failures: Authentication errors, API rate limits, network timeouts  
    - Sticky Note2 explains the purpose of this node and authorization details.

#### 2.2 Message Type Routing

- **Overview:**  
  Categorizes incoming messages by type and content using a Switch node to route to appropriate processing blocks.

- **Nodes Involved:**  
  - Switch  
  - Sticky Note5 (documentation)

- **Node Details:**  
  - **Switch**  
    - Type: Switch  
    - Config:  
      - Routes messages starting with "T " to "Task" output  
      - Routes plain text messages to "text" output  
      - Routes images to "img" output  
      - Routes audio messages to "audio" output  
      - All others to "else" output  
    - Key Expressions: Uses expressions to check message type and text prefix  
    - Input: From Line Loading Animation  
    - Output: Multiple outputs for downstream branching  
    - Potential Failures: Expression errors if message structure changes, unrecognized message types  
    - Sticky Note5 describes the router's purpose.

#### 2.3 Text Message Processing

- **Overview:**  
  Saves plain text messages (not starting with "T ") into a Microsoft Teams channel and sends a confirmation reply to the user.

- **Nodes Involved:**  
  - Microsoft Teams  
  - Line Reply (Text)  
  - Sticky Note6 (MS Teams documentation)  
  - Sticky Note3 (Line reply documentation)

- **Node Details:**  
  - **Microsoft Teams**  
    - Type: Microsoft Teams node  
    - Config: Posts the message text (converted to HTML line breaks) into a specific Teams channel "Notes" under team "Zac&Lin"  
    - Credentials: OAuth2 for Microsoft Teams  
    - Input: From Switch node ("text" output)  
    - Output: Passes data to Line Reply (Text)  
    - Potential Failures: OAuth token expiration, API permission issues

  - **Line Reply (Text)**  
    - Type: HTTP Request  
    - Config: POST to LINE `/v2/bot/message/reply` endpoint with replyToken and confirmation message "[ Message Saved in Zac&Lin > Notes ]"  
    - Authentication: HTTP Header Auth with LINE token  
    - Input: From Microsoft Teams node  
    - Output: Ends workflow branch  
    - Potential Failures: Authentication errors, invalid replyToken, network issues

#### 2.4 Task Creation from Text Starting with "T "

- **Overview:**  
  Creates a task in Microsoft To Do from text messages starting with "T " and sends a confirmation reply.

- **Nodes Involved:**  
  - Microsoft To Do  
  - Line Reply (Text)1  
  - Sticky Note4 (Tasks documentation)  
  - Sticky Note (Line reply task confirmation)

- **Node Details:**  
  - **Microsoft To Do**  
    - Type: Microsoft To Do node  
    - Config: Creates a new task with title derived from the message text after removing the "T " prefix, in a predefined task list  
    - Credentials: OAuth2 for Microsoft To Do  
    - Input: From Switch node ("Task" output)  
    - Output: Passes data to Line Reply (Text)1  
    - Potential Failures: OAuth token issues, invalid task list ID

  - **Line Reply (Text)1**  
    - Type: HTTP Request  
    - Config: POST to LINE reply endpoint with confirmation message including task title  
    - Authentication: HTTP Header Auth with LINE token  
    - Input: From Microsoft To Do node  
    - Output: Ends workflow branch  
    - Potential Failures: Same as Line Reply (Text)

#### 2.5 Image Processing and Identification

- **Overview:**  
  Downloads images sent via LINE, then uses an AI agent (OpenRouter via LangChain) to classify the image as a namecard, handwritten/text note, or other.

- **Nodes Involved:**  
  - Get Image  
  - OpenRouter Chat Model  
  - Image Router  
  - If namecard  
  - Sticky Note13 (Identify Image)  
  - Sticky Note15 (Router Namecard or not)

- **Node Details:**  
  - **Get Image**  
    - Type: HTTP Request  
    - Config: Downloads image content from LINE API using message ID  
    - Authentication: HTTP Header Auth with LINE token  
    - Input: From Switch node ("img" output)  
    - Output: Passes binary image data to OpenRouter Chat Model  
    - Potential Failures: Authentication errors, expired message content, network errors

  - **OpenRouter Chat Model**  
    - Type: LangChain OpenRouter AI Chat Model  
    - Config: Uses GPT-4o model to classify image content with prompt instructing to answer with "01" (namecard), "02" (text note), or "03" (others)  
    - Credentials: OpenRouter API credentials  
    - Input: Receives image binary data from Get Image  
    - Output: Passes classification result to Image Router node  
    - Potential Failures: API quota limits, malformed prompt, response parsing errors

  - **Image Router**  
    - Type: LangChain Agent  
    - Config: Routes based on AI output string  
    - Input: From OpenRouter Chat Model  
    - Output: Passes to If namecard node  
    - Potential Failures: Misclassification, missing output

  - **If namecard**  
    - Type: If node  
    - Config: Checks if AI output equals "01" (namecard)  
    - Input: From Image Router  
    - Output:  
      - True: Proceeds to Get Image3 (namecard extraction)  
      - False: Proceeds to Microsoft OneDrive (upload image)  
    - Potential Failures: Expression errors if AI output format changes

#### 2.6 Namecard Data Extraction and Task Creation

- **Overview:**  
  Extracts structured data from namecard images using OpenRouter.ai, creates follow-up tasks in Microsoft To Do, uploads images, and sends confirmation replies.

- **Nodes Involved:**  
  - Get Image3  
  - NamecardExtract (LangChain agent)  
  - Structured Output Parser  
  - Microsoft To Do1  
  - Line Reply Namecard  
  - HTTP Request (to external webhook)  
  - Microsoft OneDrive  
  - Microsoft OneDrive1  
  - Sticky Note14 (Namecard Extraction)  
  - Sticky Note7 (Tasks follow-up)  
  - Sticky Note10 (HTTP Request for Excel)  
  - Sticky Note8 (Line Reply for namecard)  
  - Sticky Note12 (MS Teams for namecard)  
  - Sticky Note17 (OneDrive upload explanation)

- **Node Details:**  
  - **Get Image3**  
    - Same as Get Image but used here for namecard image retrieval  
    - Input: From If namecard (true branch)  
    - Output: Passes image binary to NamecardExtract

  - **NamecardExtract**  
    - Type: LangChain Agent  
    - Config: Uses GPT-4o model to extract namecard fields in JSON format (Nickname, FirstName, LastName, CompanyFull, Department, JobTitle, Mobile, Mobile2, Email, SocialMedia, Address, Remark, NameTH)  
    - Input: Image binary from Get Image3  
    - Output: Passes JSON output to Structured Output Parser and Microsoft To Do1  
    - Potential Failures: OCR errors, incomplete extraction, API errors

  - **Structured Output Parser**  
    - Type: Output Parser for structured JSON  
    - Config: Parses AI output to JSON schema as defined in NamecardExtract  
    - Input: From NamecardExtract  
    - Output: Provides structured JSON for downstream nodes

  - **Microsoft To Do1**  
    - Type: Microsoft To Do node  
    - Config: Creates a follow-up task titled "Follow-up Namecard {{Email}}" in predefined task list  
    - Input: From NamecardExtract output  
    - Output: Passes to Line Reply Namecard  
    - Potential Failures: OAuth issues, task creation errors

  - **Line Reply Namecard**  
    - Type: HTTP Request  
    - Config: Replies to LINE user with extracted namecard JSON content  
    - Input: From Microsoft To Do1  
    - Output: Passes to HTTP Request node

  - **HTTP Request**  
    - Type: HTTP Request  
    - Config: Sends extracted namecard data and message ID to external webhook (Make.com) for further processing (e.g., adding rows in MS Excel 365)  
    - Input: From Line Reply Namecard  
    - Output: Ends branch  
    - Potential Failures: External webhook unavailability, network errors

  - **Microsoft OneDrive**  
    - Type: Microsoft OneDrive node  
    - Config: Uploads image file with temporary name "testtest.jpg" to specific OneDrive folder  
    - Input: From If namecard (false branch)  
    - Output: Passes to Microsoft OneDrive1

  - **Microsoft OneDrive1**  
    - Type: Microsoft OneDrive node  
    - Config: Renames uploaded file to LINE message ID with ".jpg" extension (workaround for known bug)  
    - Input: From Microsoft OneDrive  
    - Output: Passes to Get Image2

  - **Get Image2**  
    - Type: HTTP Request  
    - Config: Downloads renamed image from LINE API for further processing  
    - Input: From Microsoft OneDrive1  
    - Output: Passes to Other Images node

#### 2.7 Other Image Content Processing

- **Overview:**  
  For images not identified as namecards, extracts text or descriptions, uploads image to OneDrive, posts message with image in Microsoft Teams, and sends confirmation reply.

- **Nodes Involved:**  
  - Other Images (LangChain agent)  
  - OpenRouter Chat Model2  
  - Microsoft Teams1  
  - Line Reply (image)  
  - Sticky Note16 (Text Extraction)  
  - Sticky Note12 (MS Teams)  
  - Sticky Note11 (Line Reply)  
  - Sticky Note17 (OneDrive upload explanation)

- **Node Details:**  
  - **Other Images**  
    - Type: LangChain Agent  
    - Config: Extracts text if handwritten or text on screen (Thai or English), otherwise describes the image  
    - Input: From Get Image2  
    - Output: Passes extracted text to Microsoft Teams1  
    - Potential Failures: OCR errors, language detection errors, API errors

  - **Microsoft Teams1**  
    - Type: Microsoft Teams node  
    - Config: Posts extracted text with HTML line breaks and embeds the image (using OneDrive download URL) into Teams "Notes" channel  
    - Input: From Other Images  
    - Output: Passes to Line Reply (image)  
    - Potential Failures: OAuth issues, permission errors

  - **Line Reply (image)**  
    - Type: HTTP Request  
    - Config: Sends confirmation reply "[ Message Saved in Zac&Lin > Notes ]" to LINE user  
    - Input: From Microsoft Teams1  
    - Output: Ends branch  
    - Potential Failures: Authentication errors, replyToken expiration

#### 2.8 Unsupported Message Handling

- **Overview:**  
  For unsupported message types (e.g., audio), sends polite replies indicating the message type is not supported.

- **Nodes Involved:**  
  - Line Reply (Not Supported 1)  
  - Line Reply (Not Supported 2)  
  - Sticky Note9 (Line Reply unsupported message)

- **Node Details:**  
  - **Line Reply (Not Supported 1)** and **Line Reply (Not Supported 2)**  
    - Type: HTTP Request  
    - Config: POST to LINE reply endpoint with text message "Please try again. Message type is not supported"  
    - Authentication: HTTP Header Auth with LINE token (different credentials for each node)  
    - Input: From Switch node outputs "audio" and "else" respectively  
    - Output: Ends branches  
    - Potential Failures: Authentication errors, replyToken invalidation

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                                   | Input Node(s)               | Output Node(s)                | Sticky Note                                                                                              |
|-------------------------|-----------------------------------|-------------------------------------------------|-----------------------------|------------------------------|--------------------------------------------------------------------------------------------------------|
| Line Webhook            | Webhook                           | Receive incoming LINE messages                    | —                           | Line Loading Animation        | You need to set-up this webhook at Line Manager or Line Developer Console. Copy Webhook URL here. https://developers.line.biz/en/docs/messaging-api/receiving-messages/ |
| Line Loading Animation  | HTTP Request                     | Send loading animation feedback to LINE user     | Line Webhook                | Switch                       | This node sends loading animation to user to indicate processing. https://developers.line.biz/en/docs/messaging-api/use-loading-indicator/ |
| Switch                  | Switch                           | Route message by type and content prefix          | Line Loading Animation      | Microsoft To Do, Microsoft Teams, Get Image, Line Reply (Not Supported 1), Line Reply (Not Supported 2) | Router for Tasks (Text started with 'T'), other texts, images and others                                |
| Microsoft To Do         | Microsoft To Do                  | Create task from text starting with "T "          | Switch ("Task" output)      | Line Reply (Text)1           | Tasks: To add in MS 'To Do' List                                                                       |
| Line Reply (Text)1      | HTTP Request                    | Reply confirmation for task creation               | Microsoft To Do             | —                            | Line Reply: To send feedback that the task has been added                                              |
| Microsoft Teams         | Microsoft Teams                 | Save plain text message into Teams channel         | Switch ("text" output)      | Line Reply (Text)            | MS Teams: Save this message in MS Teams                                                                |
| Line Reply (Text)       | HTTP Request                    | Reply confirmation for message saved in Teams      | Microsoft Teams             | —                            | Line Reply: To send feedback message has been saved                                                    |
| Get Image               | HTTP Request                    | Download image content from LINE                    | Switch ("img" output)       | OpenRouter Chat Model        |                                                                                                        |
| OpenRouter Chat Model   | LangChain OpenRouter AI Chat    | Classify image as namecard, text note, or other    | Get Image                  | Image Router                 |                                                                                                        |
| Image Router            | LangChain Agent                | Route based on image classification                 | OpenRouter Chat Model       | If namecard                  |                                                                                                        |
| If namecard             | If                             | Check if image is namecard                           | Image Router                | Get Image3 (true), Microsoft OneDrive (false) | Router Namecard or not                                                                                   |
| Get Image3              | HTTP Request                    | Download image content for namecard extraction      | If namecard (true)          | NamecardExtract              |                                                                                                        |
| NamecardExtract         | LangChain Agent                | Extract structured data from namecard image         | Get Image3                  | Structured Output Parser, Microsoft To Do1 | Namecard Information Extraction                                                                         |
| Structured Output Parser| LangChain Output Parser         | Parse AI output into structured JSON                 | NamecardExtract             | NamecardExtract              |                                                                                                        |
| Microsoft To Do1        | Microsoft To Do                  | Create follow-up task from extracted namecard data  | NamecardExtract             | Line Reply Namecard          | Tasks: To add in MS 'To Do' List to follow-up with this namecard                                       |
| Line Reply Namecard     | HTTP Request                    | Reply with extracted namecard data                   | Microsoft To Do1            | HTTP Request                | Line Reply: To send feedback message has been saved                                                    |
| HTTP Request            | HTTP Request                    | Send extracted namecard data to external webhook    | Line Reply Namecard         | —                            | HTTP Request: This is to trigger another workflow to add new rows in MS Excel 365                      |
| Microsoft OneDrive      | Microsoft OneDrive              | Upload image to OneDrive                             | If namecard (false)         | Microsoft OneDrive1          | Upload to OneDrive. Due to some bug, file needs renaming                                              |
| Microsoft OneDrive1     | Microsoft OneDrive              | Rename uploaded image file                           | Microsoft OneDrive          | Get Image2                   |                                                                                                        |
| Get Image2              | HTTP Request                    | Download renamed image                               | Microsoft OneDrive1         | Other Images                |                                                                                                        |
| Other Images            | LangChain Agent                | Extract text or describe image content               | Get Image2                  | Microsoft Teams1             | Text Extraction                                                                                         |
| Microsoft Teams1        | Microsoft Teams                 | Post extracted text and image in Teams channel       | Other Images                | Line Reply (image)           | MS Teams: Save this message in MS Teams                                                                |
| Line Reply (image)      | HTTP Request                    | Reply confirmation for image message saved           | Microsoft Teams1            | —                            | Line Reply: To send feedback message has been saved                                                    |
| Line Reply (Not Supported 1) | HTTP Request              | Reply for unsupported message types                   | Switch ("audio" output)     | —                            | Line Reply: To reply that message is not supported                                                    |
| Line Reply (Not Supported 2) | HTTP Request              | Reply for other unsupported message types             | Switch ("else" output)      | —                            | Line Reply: To reply that message is not supported                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `minibear`  
   - Purpose: Receive incoming messages from LINE platform.

2. **Create HTTP Request Node for Loading Animation**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.line.me/v2/bot/chat/loading/start`  
   - Body (JSON):  
     ```json
     {
       "chatId": "{{ $json.body.events[0].source.userId }}",
       "loadingSeconds": 60
     }
     ```  
   - Authentication: HTTP Header Auth with LINE Bot token  
   - Connect Webhook node output to this node.

3. **Add Switch Node to Route Messages**  
   - Type: Switch  
   - Rules:  
     - Output "Task": If message text starts with "T "  
     - Output "text": If message type equals "text"  
     - Output "img": If message type equals "image"  
     - Output "audio": If message type equals "audio"  
     - Output "else": All other cases  
   - Connect Loading Animation node output to Switch node input.

4. **Task Creation Branch (Text Starting with "T ")**  
   - Add Microsoft To Do node:  
     - Operation: Create task  
     - Task title: `{{$json.body.events[0].message.text.replace('T ', '')}}`  
     - Task list ID: Your Microsoft To Do list ID  
     - Credentials: Microsoft To Do OAuth2  
   - Add HTTP Request node to reply to LINE:  
     - POST to `https://api.line.me/v2/bot/message/reply`  
     - Body:  
       ```json
       {
         "replyToken": "{{ $json.body.events[0].replyToken }}",
         "messages": [{"type": "text", "text": "[ Task : {{ $json.body.events[0].message.text.replace('T ', '') }} created successfully in Private Task ]"}]
       }
       ```  
     - Authentication: HTTP Header Auth with LINE token  
   - Connect Switch "Task" output → Microsoft To Do → Line Reply (Text)1

5. **Plain Text Message Branch**  
   - Add Microsoft Teams node:  
     - Team ID and Channel ID: Your target Teams channel  
     - Message: `{{$json.body.events[0].message.text.replace('\n\n', '<br><br>').replace('\n', '<br>')}}`  
     - Content type: HTML  
     - Credentials: Microsoft Teams OAuth2  
   - Add HTTP Request node to reply to LINE with confirmation message  
   - Connect Switch "text" output → Microsoft Teams → Line Reply (Text)

6. **Image Message Branch**  
   - Add HTTP Request node to download image content from LINE:  
     - URL: `https://api-data.line.me/v2/bot/message/{{ $json.body.events[0].message.id }}/content`  
     - Authentication: HTTP Header Auth with LINE token  
   - Add LangChain OpenRouter Chat Model node:  
     - Model: GPT-4o  
     - Prompt: "You'll identify the image\n01 Namecard\n02 Text on screen or handwritten note\n03 Others\nYou'll answer with only 01 02 or 03"  
     - Pass binary image data  
     - Credentials: OpenRouter API  
   - Add LangChain Agent node (Image Router) to route based on AI output  
   - Add If node to check if output equals "01" (namecard)  
   - Connect Switch "img" output → Get Image → OpenRouter Chat Model → Image Router → If namecard

7. **Namecard True Branch**  
   - Add HTTP Request node to download image again (Get Image3)  
   - Add LangChain Agent node (NamecardExtract) with prompt to extract structured JSON fields from namecard image  
   - Add Structured Output Parser node with JSON schema for namecard fields  
   - Add Microsoft To Do node to create follow-up task titled "Follow-up Namecard {{Email}}"  
   - Add HTTP Request node to reply to LINE with extracted namecard JSON  
   - Add HTTP Request node to send extracted data to external webhook (e.g., Make.com)  
   - Connect If namecard true → Get Image3 → NamecardExtract → Structured Output Parser → Microsoft To Do1 → Line Reply Namecard → HTTP Request

8. **Namecard False Branch (Other Images)**  
   - Add Microsoft OneDrive node to upload image (temporary filename)  
   - Add Microsoft OneDrive node to rename uploaded file to message ID + ".jpg"  
   - Add HTTP Request node to download renamed image (Get Image2)  
   - Add LangChain Agent node (Other Images) with prompt to extract text or describe image  
   - Add Microsoft Teams node to post extracted text and embed image URL from OneDrive  
   - Add HTTP Request node to reply to LINE with confirmation  
   - Connect If namecard false → Microsoft OneDrive → Microsoft OneDrive1 → Get Image2 → Other Images → Microsoft Teams1 → Line Reply (image)

9. **Audio and Unsupported Message Branch**  
   - Add two HTTP Request nodes to reply with "Please try again. Message type is not supported"  
   - Connect Switch "audio" output → Line Reply (Not Supported 1)  
   - Connect Switch "else" output → Line Reply (Not Supported 2)

10. **Credential Setup**  
    - Configure HTTP Header Auth credentials with LINE Bot tokens for all LINE API calls  
    - Configure Microsoft OAuth2 credentials for Teams, To Do, and OneDrive nodes  
    - Configure OpenRouter API credentials for LangChain nodes

11. **Test and Deploy**  
    - Set webhook URL in LINE Developer Console to your n8n webhook URL with path `/minibear`  
    - Test sending text, tasks (starting with "T "), images (namecards and others), and unsupported types  
    - Monitor workflow executions and logs for errors and adjust as needed

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                                                                          |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| You need to set-up this webhook at Line Manager or Line Developer Console. Copy Webhook URL from the webhook node and remove 'test' part when in production. | https://developers.line.biz/en/docs/messaging-api/receiving-messages/                                   |
| Loading animation node sends feedback to users that the workflow is running to avoid user uncertainty.                                                        | https://developers.line.biz/en/docs/messaging-api/use-loading-indicator/                                |
| This workflow uses OpenRouter.ai GPT-4o model for AI-powered image classification and data extraction.                                                        | https://openrouter.ai/                                                                                   |
| Microsoft Teams and Microsoft To Do integrations require OAuth2 credentials with appropriate permissions.                                                     | https://learn.microsoft.com/en-us/graph/api/resources/microsoft-graph-overview                            |
| External HTTP Request node sends extracted namecard data to a Make.com webhook for further processing (e.g., Excel row insertion).                           | https://www.make.com/en/integrations/http                                                                 |

---

This documentation provides a complete and structured understanding of the workflow, enabling advanced users or AI agents to reproduce, modify, and troubleshoot the workflow effectively.