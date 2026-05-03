Process Telegram Image Albums with Data Tables Cache and NanoBanana AI

https://n8nworkflows.xyz/workflows/process-telegram-image-albums-with-data-tables-cache-and-nanobanana-ai-9288


# Process Telegram Image Albums with Data Tables Cache and NanoBanana AI

### 1. Workflow Overview

This workflow automates the processing of Telegram image albums (media groups) using a data table cache and AI analysis via NanoBanana through OpenRouter. It targets scenarios where users send albums of images with optional captions to a Telegram bot, and the workflow processes these albums collectively to generate AI-assisted responses, returning a summarized or enhanced media output to the user.

The workflow is logically divided into these blocks:

- **1.1 Telegram Input Reception and Storage**: Captures Telegram messages with media groups, filters for albums with images, and stores incoming messages in a data table with statuses for processing control.
- **1.2 Scheduled Polling for New Media Groups**: Periodically checks the data table for new media group records to process.
- **1.3 Media Group Aggregation and Preparation**: Collects all messages belonging to the same media group from the data table, sorts and prepares them with captions and image URLs for AI processing.
- **1.4 AI Processing via NanoBanana (OpenRouter)**: Sends aggregated media group data (images and captions) to NanoBanana AI model for summarization or analysis.
- **1.5 Post-AI Processing and Telegram Response**: Extracts AI results, converts image data to binary, sends results back to the Telegram chat, and updates data table statuses accordingly.
- **1.6 User Notifications and Workflow Controls**: Sends user notifications at different stages and manages workflow triggers including manual and schedule triggers.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Input Reception and Storage

- **Overview:**  
  This block listens for all Telegram updates, filters for media groups containing images, and inserts or updates records in a data table to track the processing status of each media group message.

- **Nodes Involved:**  
  Telegram Trigger, Is media group with images? (IF), Upsert row(s) (Data Table), Filter1, Send initial notification, Process other messages as usual (NOOP), Sticky Notes (contextual)

- **Node Details:**

  - **Telegram Trigger**  
    - Type: Telegram Trigger  
    - Role: Listens to all Telegram updates for the bot  
    - Config: Receives all update types  
    - Credentials: Telegram API for bot token  
    - Outputs: Sends raw Telegram message JSON downstream  
    - Edge cases: Possible Telegram API downtime or webhook issues

  - **Is media group with images? (IF Node)**  
    - Type: IF  
    - Role: Checks if incoming message is part of a media group with photos  
    - Config:  
      - Condition 1: media_group_id is not empty  
      - Condition 2: message.photo array exists  
    - Outputs:  
      - True: Proceed to upsert row and further processing  
      - False: Moves to NOOP for other message types  
    - Edge cases: Messages without media_group_id or non-photo messages

  - **Upsert row(s) (Data Table)**  
    - Type: Data Table  
    - Role: Inserts or updates a record in the data table to cache message details  
    - Config:  
      - Columns: status (default "new"), chat_id, message_id, media_group, full message JSON string  
      - Filter: Matches on chat_id and message_id for upsert  
      - Data Table: TG_mediagroup  
    - Edge cases: Data table write failures, JSON parsing errors

  - **Filter1**  
    - Type: Filter  
    - Role: Checks if the message has a caption (not empty)  
    - Config: Condition to check non-empty caption string  
    - Outcome: If caption exists, sends a notification message back

  - **Send initial notification**  
    - Type: Telegram  
    - Role: Sends a reply notification to user that processing will begin soon  
    - Config: Sends text reply linked to original message_id  
    - Credentials: Same Telegram bot credentials

  - **Process other messages as usual (NOOP)**  
    - Type: NoOp  
    - Role: Pass-through for messages not part of media groups with photos  
    - No configuration

  - **Sticky Notes:**  
    - Explain assumptions about captions and user notification  
    - Provide prerequisites about data table and Telegram bot setup  
    - Link to LinkedIn and offer branding

---

#### 1.2 Scheduled Polling for New Media Groups

- **Overview:**  
  A schedule trigger periodically activates to check the data table for any new media group messages that require processing.

- **Nodes Involved:**  
  Schedule Trigger, If any new records (Data Table), Get new requests (Data Table), NOOP, Merge2, Sticky Note4

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Fires every 5 seconds to initiate processing cycles  
    - Config: Interval of 5 seconds

  - **If any new records (Data Table)**  
    - Type: Data Table (rowExists)  
    - Role: Checks if any records with status "new" exist in the data table  
    - Config: Filter on status = "new"  
    - Outputs:  
      - True: Proceed to get new requests  
      - False: Stops

  - **Get new requests (Data Table)**  
    - Type: Data Table (get)  
    - Role: Retrieves all records with status "new" for processing  
    - Config: Filter on status = "new", return all records

  - **NOOP**  
    - Type: NoOp  
    - Role: Pass-through node to continue workflow

  - **Merge2**  
    - Type: Merge  
    - Role: Combines branches for status update later

  - **Sticky Note4**  
    - Notes the timer is adjustable for checking new records

---

#### 1.3 Media Group Aggregation and Preparation

- **Overview:**  
  This block merges and sorts media group messages to prepare aggregated data including captions and image URLs, ready to be sent for AI processing.

- **Nodes Involved:**  
  status:processing (Data Table update), Sort, parse TG messages (Set), Get a file (Telegram), prepare user messages (Set), Summarize, Send processing message, Sticky Note1, Sticky Note2

- **Node Details:**

  - **status:processing (Data Table)**  
    - Type: Data Table (update)  
    - Role: Updates the media group record status to "processing"  
    - Config: Updates by record ID from incoming data

  - **Sort**  
    - Type: Sort  
    - Role: Sorts records by media_group and createdAt to ensure ordered processing

  - **parse TG messages (Set)**  
    - Type: Set  
    - Role: Parses the JSON string from the message field into an object  
    - Config: JSON.parse on message string field

  - **Get a file (Telegram)**  
    - Type: Telegram node  
    - Role: Retrieves file metadata from Telegram using the file_id of the highest resolution photo in the message  
    - Config: Uses last photo's file_id, does not download file, just fetches file info  
    - Credentials: Telegram API

  - **prepare user messages (Set)**  
    - Type: Set  
    - Role: Prepares structured objects for AI input:  
      - text: caption if exists  
      - img: constructs image_url object with direct Telegram file download URL (requires manual token replacement)  
      - media_group and chat_id for reference  
    - Config: Uses expression to build URL:  
      `https://api.telegram.org/file/bot<TOKEN>/${file_path}` (TOKEN must be replaced by user)

  - **Summarize**  
    - Type: Summarize  
    - Role: Aggregates the prepared messages grouped by chat_id and media_group, appending all image and text fields into arrays for AI input

  - **Send processing message**  
    - Type: Telegram  
    - Role: Notifies user that processing is underway

  - **Sticky Notes:**  
    - Explain delay check to protect against slow internet  
    - Provide instruction to update Telegram file download URL in "prepare user messages" node

---

#### 1.4 AI Processing via NanoBanana (OpenRouter)

- **Overview:**  
  This block sends the aggregated media group data (images and captions) to the NanoBanana AI model hosted on OpenRouter for multi-modal processing.

- **Nodes Involved:**  
  Call NanoBanana via OpenRouter (HTTP Request), Extract Base64 (Set), Convert to File, Merge1, Sticky Note3

- **Node Details:**

  - **Call NanoBanana via OpenRouter (HTTP Request)**  
    - Type: HTTP Request  
    - Role: Sends JSON body to OpenRouter API to invoke NanoBanana AI with images and text  
    - Config:  
      - URL: https://openrouter.ai/api/v1/chat/completions  
      - Method: POST  
      - Body: JSON including model "google/gemini-2.5-flash-image-preview"  
      - Messages: Concatenation of text and images arrays from previous step  
      - Modalities: ["image", "text"]  
      - Authentication: OpenRouter API credentials  
    - Edge cases: API rate limits, network errors, invalid API keys

  - **Extract Base64 (Set)**  
    - Type: Set  
    - Role: Extracts and isolates the base64-encoded image URL returned by NanoBanana from the AI response  
    - Config: Parses response JSON to extract base64 string after comma

  - **Convert to File**  
    - Type: ConvertToFile  
    - Role: Converts the base64 string into binary for Telegram to send as a photo

  - **Merge1**  
    - Type: Merge  
    - Role: Combines the binary file data with other message fields for sending

  - **Sticky Note3:**  
    - Explains sending array with caption and images to NanoBanana and transforming LLM output to binary

---

#### 1.5 Post-AI Processing and Telegram Response

- **Overview:**  
  Sends the AI-generated image back to the user in Telegram, updates the data table status to "done," and manages message replies.

- **Nodes Involved:**  
  Send result message (Telegram), status:done (Data Table), Merge2

- **Node Details:**

  - **Send result message (Telegram)**  
    - Type: Telegram  
    - Role: Sends the processed photo back to the Telegram chat as a reply to the original message  
    - Config:  
      - chatId from AI response context  
      - operation: sendPhoto  
      - binaryData: true (sends the binary image)  
      - reply_to_message_id to link response to original message  
    - Credentials: Telegram API

  - **status:done (Data Table)**  
    - Type: Data Table (update)  
    - Role: Updates the record status to "done" to mark processing complete  
    - Config: Updates by record ID

  - **Merge2**  
    - Type: Merge  
    - Role: Synchronizes update flows for status changes

---

#### 1.6 User Notifications and Workflow Controls

- **Overview:**  
  Handles manual triggers for testing, merges data flows, and manages timing and execution policies for the workflow.

- **Nodes Involved:**  
  When clicking ‘Execute workflow’ (Manual Trigger), various Merge and NOOP nodes, Sticky Notes

- **Node Details:**

  - **When clicking ‘Execute workflow’ (Manual Trigger)**  
    - Type: Manual Trigger  
    - Role: Allows manual execution of the workflow for testing or instant processing  
    - Config: No parameters

  - **Merge and NOOP nodes**  
    - Used for flow control, combining branches, or bypassing certain paths

  - **Sticky Notes**  
    - Provide branding, LinkedIn link, prerequisites, and instructions for setup

---

### 3. Summary Table

| Node Name                   | Node Type               | Functional Role                             | Input Node(s)                | Output Node(s)               | Sticky Note                                                  |
|-----------------------------|-------------------------|---------------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------|
| Telegram Trigger             | Telegram Trigger        | Receives Telegram updates                    |                             | Is media group with images?  |                                                              |
| Is media group with images?  | IF                      | Filters messages for media groups with photos | Telegram Trigger            | Upsert row(s), Filter1, Process other messages as usual |                                                              |
| Upsert row(s)               | Data Table              | Inserts/updates message records in data table | Is media group with images? |                             |                                                              |
| Filter1                     | Filter                  | Checks for non-empty caption                  | Upsert row(s)               | Send initial notification    |                                                              |
| Send initial notification    | Telegram                | Notifies user processing will begin soon    | Filter1                     |                             |                                                              |
| Process other messages as usual | NoOp                 | Pass-through for non-media group messages    | Is media group with images? |                             |                                                              |
| Schedule Trigger             | Schedule Trigger        | Triggers periodic polling                     |                             | If any new records           | Adjust timer for checking the new records in Data Tables     |
| If any new records           | Data Table (rowExists)  | Checks for new "new" status records           | Schedule Trigger            | Get new requests             |                                                              |
| Get new requests             | Data Table (get)        | Fetches all new records                        | If any new records          | NOOP, Merge2                |                                                              |
| NOOP                        | NoOp                    | Pass-through node                             | Get new requests            | Merge                        |                                                              |
| Merge                       | Merge                   | Combines flow to update status                 | NOOP                       | Latest message in media_group |                                                              |
| Latest message in media_group | Summarize              | Aggregates max createdAt per media group       | Merge                       | Filter                      | Check if latest media group message has sufficient delay     |
| Filter                      | Filter                  | Filters messages older than 7 seconds          | Latest message in media_group | Merge                     |                                                              |
| status:processing           | Data Table (update)     | Updates record status to "processing"          | Merge                       | Sort                        |                                                              |
| Sort                        | Sort                    | Sorts messages by media_group and createdAt    | status:processing           | parse TG messages           |                                                              |
| parse TG messages           | Set                     | Parses stored message JSON                      | Sort                        | Get a file                  |                                                              |
| Get a file                  | Telegram                | Retrieves file metadata for highest-res photo | parse TG messages           | prepare user messages       | When getting file URLs directly from Telegram, update download link |
| prepare user messages       | Set                     | Prepares image and text data for AI             | Get a file                  | Summarize                   | Update Telegram file download URL in this node               |
| Summarize                  | Summarize                | Aggregates images and captions for AI input    | prepare user messages       | Send processing message, Call NanoBanana via OpenRouter |                                                              |
| Send processing message     | Telegram                | Notifies user that processing is underway       | Summarize                   | Merge1                      |                                                              |
| Call NanoBanana via OpenRouter | HTTP Request         | Sends data to NanoBanana AI model via OpenRouter | Summarize                  | Extract Base64              | Send array with caption text and all images to NanoBanana via OpenRouter |
| Extract Base64             | Set                      | Extracts base64 image string from AI response   | Call NanoBanana via OpenRouter | Convert to File           |                                                              |
| Convert to File            | ConvertToFile             | Converts base64 string to binary file            | Extract Base64              | Merge1                      |                                                              |
| Merge1                     | Merge                     | Combines AI output file with message data        | Send processing message, Convert to File | Send result message  |                                                              |
| Send result message        | Telegram                  | Sends AI-generated photo back to user            | Merge1                      | Merge2                      |                                                              |
| Merge2                     | Merge                     | Merges flow to update status to "done"           | Send result message, Get new requests | status:done          |                                                              |
| status:done                | Data Table (update)       | Updates record status to "done"                    | Merge2                      |                             |                                                              |
| When clicking ‘Execute workflow’ | Manual Trigger        | Allows manual workflow execution                   |                             | If any new records          |                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Data Table**  
   - Create a new Data Table named e.g. `TG_mediagroup`  
   - Columns (all string type): `chat_id`, `message_id`, `media_group`, `message`, `status`

2. **Setup Telegram Credentials**  
   - Create a new Telegram bot via @BotFather  
   - Copy bot token and create Telegram credentials in n8n

3. **Create Telegram Trigger Node**  
   - Type: Telegram Trigger  
   - Credentials: Telegram bot credentials  
   - Updates: All (`*`)  
   - Position: Left/top area

4. **Add IF Node: Is media group with images?**  
   - Check: `media_group_id` is not empty  
   - Check: `message.photo` array exists  
   - True branch: Continue; False branch: NOOP node for other messages

5. **Add Data Table Node: Upsert row(s)**  
   - Operation: Upsert  
   - Data Table: `TG_mediagroup`  
   - Filter columns: `chat_id` and `message_id` from incoming message  
   - Columns to insert/update:  
     - status = "new"  
     - chat_id, message_id, media_group from message  
     - message as JSON stringified full message

6. **Add Filter Node: Filter1**  
   - Condition: message caption is not empty

7. **Add Telegram Node: Send initial notification**  
   - Text: "Received your message, processing will begin shortly"  
   - chatId: message.chat.id  
   - reply_to_message_id: message.message_id

8. **Add NOOP Node: Process other messages as usual**  
   - For messages not part of media groups

9. **Add Schedule Trigger Node**  
   - Interval: Every 5 seconds

10. **Add Data Table Node: If any new records (rowExists)**  
    - Filter: status = "new"

11. **Add Data Table Node: Get new requests (get all)**  
    - Filter: status = "new"  
    - Return all results

12. **Add NOOP Node and Merge2 Node**  
    - For flow control and synchronization

13. **Add Data Table Node: status:processing (update)**  
    - Update record status to "processing" by record ID

14. **Add Sort Node**  
    - Sort by `media_group` and `createdAt`

15. **Add Set Node: parse TG messages**  
    - Parse `message` field JSON string into an object

16. **Add Telegram Node: Get a file**  
    - Use last photo's file_id from message.photo array  
    - Resource: file  
    - Download: false  
    - Credentials: Telegram bot

17. **Add Set Node: prepare user messages**  
    - Assign:  
      - `text`: caption if exists else null  
      - `img`: construct object with `image_url.url` as `https://api.telegram.org/file/bot<TOKEN>/${file_path}` (replace `<TOKEN>` manually)  
      - `media_group`, `chat_id`

18. **Add Summarize Node**  
    - Group by `chat_id` and `media_group`  
    - Append fields: `img` and `text`

19. **Add Telegram Node: Send processing message**  
    - Text: "Processing your request, please wait..."  
    - chatId: from aggregated data

20. **Add HTTP Request Node: Call NanoBanana via OpenRouter**  
    - URL: `https://openrouter.ai/api/v1/chat/completions`  
    - Method: POST  
    - Authentication: OpenRouter API credentials  
    - Body JSON:  
      ```
      {
        "model": "google/gemini-2.5-flash-image-preview",
        "messages": [
          {
            "role": "user",
            "content": $json.appended_text.concat($json.appended_img)
          }
        ],
        "modalities": ["image", "text"]
      }
      ```
    - Batch size: 1, interval 300 ms

21. **Add Set Node: Extract Base64**  
    - Extract base64 portion from AI response image URL

22. **Add ConvertToFile Node: Convert to File**  
    - Convert base64 string to binary file

23. **Add Merge Node: Merge1**  
    - Combine processing message and binary file data

24. **Add Telegram Node: Send result message**  
    - Operation: sendPhoto  
    - chatId: from AI response  
    - binaryData: true  
    - reply_to_message_id: original message ID

25. **Add Data Table Node: status:done (update)**  
    - Update record status to "done"

26. **Connect nodes following the logic in section 1**

27. **Add Sticky Notes**  
    - Add notes on prerequisites, URL update instructions, and LinkedIn contact

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| Create Data Table with columns: chat_id, message_id, media_group, message, status                                    | Prerequisite for caching Telegram messages                                                                         |
| Use @BotFather to create a Telegram bot and set up Telegram credentials in n8n                                       | Telegram bot setup                                                                                                   |
| Update the file download URL in "prepare user messages" node manually replacing `<TOKEN>` with your Telegram bot token | Necessary for correct file download links from Telegram API                                                         |
| Protect from slow internet speed by checking message delay before AI processing                                       | See "Check if latest media group message has sufficient delay" sticky note                                          |
| Branding and contact: Reach out via LinkedIn: https://www.linkedin.com/in/parsadanyan/                                | [LinkedIn Profile](https://www.linkedin.com/in/parsadanyan/)                                                        |
| Video demonstration link: https://short.fyi/current-offer                                                           | Provided in sticky note for workflow promotion                                                                       |

---

**Disclaimer:**  
This documentation is generated from an n8n workflow for lawful, public data processing. It complies with all content policies and contains no illegal or protected content.