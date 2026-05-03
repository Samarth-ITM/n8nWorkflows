Track Food Calories via Telegram with GPT-4 Vision and Google Sheets

https://n8nworkflows.xyz/workflows/track-food-calories-via-telegram-with-gpt-4-vision-and-google-sheets-7006


# Track Food Calories via Telegram with GPT-4 Vision and Google Sheets

### 1. Workflow Overview

This workflow automates the process of tracking food calories via Telegram by leveraging GPT-4 Vision for image analysis and Google Sheets for logging. Users send food images through Telegram, which are then analyzed by GPT-4 Vision to describe the food and estimate calorie content. The results are logged in a Google Sheet and sent back to the user as a message.

Logical blocks:

- **1.1 Input Reception:** Captures incoming Telegram messages containing food images.
- **1.2 User Acknowledgment:** Sends an immediate acknowledgment message to the user confirming receipt.
- **1.3 Image Download and Preparation:** Downloads the food image from Telegram and prepares it in a binary format suitable for AI processing.
- **1.4 AI Processing:** Sends the image to GPT-4 Vision to generate a descriptive text with calorie estimation.
- **1.5 Data Logging and Response:** Logs the AI response into a Google Sheet and sends the analysis result back to the user on Telegram.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Listens for Telegram updates specifically for inbound messages.
- **Nodes Involved:** Telegram Trigger
- **Node Details:**
  - **Name:** Telegram Trigger
  - **Type:** Telegram Trigger node
  - **Role:** Entry point that triggers the workflow when a Telegram message is received.
  - **Configuration:** Listens to "message" updates only.
  - **Key Expressions:** Uses Telegram API credentials for authentication.
  - **Input:** Incoming Telegram message.
  - **Output:** Message object containing text, photos, chat details.
  - **Version:** 1
  - **Potential Failures:** Telegram API authentication errors, connectivity issues, unsupported update types.
  - **Sub-workflow:** None

#### 2.2 User Acknowledgment

- **Overview:** Sends a confirmation message back to the user after receiving their food image.
- **Nodes Involved:** Sticky Note: Acknowledge
- **Node Details:**
  - **Name:** Sticky Note: Acknowledge
  - **Type:** Telegram node (send message)
  - **Role:** Provides immediate user feedback with fixed text: "🧠 Thanks! I’m analyzing your food image now..."
  - **Configuration:** Sends a message to the chat ID extracted dynamically from the incoming message.
  - **Key Expressions:**
    - `chatId`: `={{$json["message"]["chat"]["id"]}}`
  - **Input:** Triggered after Telegram Trigger.
  - **Output:** Confirmation message sent to user.
  - **Version:** 1
  - **Potential Failures:** Telegram API errors, invalid chat ID, message length limits.
  - **Sub-workflow:** None

#### 2.3 Image Download and Preparation

- **Overview:** Downloads the highest resolution photo sent by the user and formats it into a binary object for AI processing.
- **Nodes Involved:** Download Telegram Image, Prepare Binary Image
- **Node Details:**

  - **Download Telegram Image**
    - **Type:** Telegram node (file download)
    - **Role:** Downloads the photo file using the last (highest resolution) photo file_id from the message.
    - **Configuration:**
      - `fileId`: Extracts the last photo's file_id dynamically.
      - `resource`: file
      - `operation`: download
    - **Key Expressions:**
      - `fileId`: `={{$json["message"]["photo"][ $json["message"]["photo"].length - 1 ]["file_id"]}}`
    - **Input:** Triggered after acknowledgment.
    - **Output:** Binary file data.
    - **Version:** 1
    - **Potential Failures:** File not found, Telegram API download limits, file permission errors.

  - **Prepare Binary Image**
    - **Type:** Function node
    - **Role:** Converts the downloaded file binary data to a named binary property `image` for compatibility with OpenAI node.
    - **Configuration:** Returns a single item with binary property `image` copied from `data`.
    - **Code:**
      ```javascript
      return [{ binary: { image: items[0].binary.data } }];
      ```
    - **Input:** Binary data from download node.
    - **Output:** Binary data with property `image`.
    - **Version:** 1
    - **Potential Failures:** Missing binary data, code execution errors.

#### 2.4 AI Processing

- **Overview:** Uses GPT-4 Vision to analyze the image and generate a description including calorie estimation.
- **Nodes Involved:** GPT-4 Vision Analyze
- **Node Details:**
  - **Name:** GPT-4 Vision Analyze
  - **Type:** OpenAI node
  - **Role:** Sends the food image to GPT-4 Vision model with a prompt to describe food and estimate calories.
  - **Configuration:**
    - `model`: `gpt-4-vision-preview`
    - `messages`: Array with a user role message containing text prompt and image URL referencing the binary image.
  - **Key Expressions:**
    - `image_url`: `={{$binary.image.data}}`
    - Prompt text: "Describe this food and estimate its calories."
  - **Input:** Binary image from Prepare Binary Image node.
  - **Output:** JSON with text analysis of food and calorie estimation.
  - **Version:** 1
  - **Potential Failures:** OpenAI API quota exceeded, invalid image format, connection timeout, unauthorized access.
  - **Sub-workflow:** None

#### 2.5 Data Logging and Response

- **Overview:** Logs the analysis results into Google Sheets and sends the text back to the Telegram user.
- **Nodes Involved:** Log to Google Sheet, Send GPT Result
- **Node Details:**

  - **Log to Google Sheet**
    - **Type:** Google Sheets node
    - **Role:** Appends a new row with the timestamp and AI text response to a specified spreadsheet.
    - **Configuration:**
      - `range`: "Sheet1!A:C"
      - `values`: Dynamic array with current ISO timestamp and the text result (`$json["text"]`).
      - `sheetId`: Set via credentials variable.
      - `authentication`: OAuth2
    - **Key Expressions:**
      - `values`: `={{ [[new Date().toISOString(), $json["text"]]] }}`
    - **Input:** AI output from GPT-4 Vision node.
    - **Output:** Confirmation of row append.
    - **Version:** 2
    - **Potential Failures:** Google API authentication errors, sheet ID invalid, quota limits, range errors.

  - **Send GPT Result**
    - **Type:** Telegram node (send message)
    - **Role:** Sends the AI-generated description and calorie estimation back to the user on Telegram.
    - **Configuration:**
      - `text`: `={{$json["text"]}}`
      - `chatId`: `={{$json["message"]["chat"]["id"]}}`
    - **Input:** AI output.
    - **Output:** Telegram message with analysis result.
    - **Version:** 1
    - **Potential Failures:** Telegram API errors, invalid chat ID, message size limits.

---

### 3. Summary Table

| Node Name              | Node Type             | Functional Role                   | Input Node(s)         | Output Node(s)                   | Sticky Note                                  |
|------------------------|-----------------------|---------------------------------|-----------------------|---------------------------------|----------------------------------------------|
| Telegram Trigger       | Telegram Trigger       | Entry point, listens for messages | -                     | Sticky Note: Acknowledge        |                                              |
| Sticky Note: Acknowledge | Telegram              | Sends acknowledgment message    | Telegram Trigger       | Download Telegram Image          |                                              |
| Download Telegram Image | Telegram              | Downloads highest resolution photo | Sticky Note: Acknowledge | Prepare Binary Image           |                                              |
| Prepare Binary Image    | Function              | Prepares binary image for AI     | Download Telegram Image | GPT-4 Vision Analyze            |                                              |
| GPT-4 Vision Analyze   | OpenAI                 | Analyzes image and estimates calories | Prepare Binary Image   | Log to Google Sheet, Send GPT Result |                                              |
| Log to Google Sheet    | Google Sheets          | Logs analysis results with timestamp | GPT-4 Vision Analyze   | -                               |                                              |
| Send GPT Result        | Telegram               | Sends AI analysis result to user | GPT-4 Vision Analyze   | -                               |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Telegram Trigger node:**
   - Type: Telegram Trigger
   - Parameters: Listen for update type "message"
   - Credentials: Link Telegram API credentials (OAuth2 or bot token)
   - Position: Start node

2. **Add a Telegram node (Sticky Note: Acknowledge):**
   - Type: Telegram (Send Message)
   - Parameters:
     - Text: "🧠 Thanks! I’m analyzing your food image now..."
     - Chat ID: `={{$json["message"]["chat"]["id"]}}`
   - Credentials: Use the same Telegram credentials as trigger
   - Connect from Telegram Trigger

3. **Add a Telegram node (Download Telegram Image):**
   - Type: Telegram (Download file)
   - Parameters:
     - File ID: `={{$json["message"]["photo"][ $json["message"]["photo"].length - 1 ]["file_id"]}}`
     - Resource: File
     - Operation: Download
   - Credentials: Same Telegram credentials
   - Connect from Sticky Note: Acknowledge

4. **Add a Function node (Prepare Binary Image):**
   - Type: Function
   - Parameters: Paste following code:
     ```javascript
     return [{ binary: { image: items[0].binary.data } }];
     ```
   - Connect from Download Telegram Image

5. **Add an OpenAI node (GPT-4 Vision Analyze):**
   - Type: OpenAI
   - Parameters:
     - Model: gpt-4-vision-preview
     - Messages: Set as an array with a single message object:
       ```json
       [
         {
           "role": "user",
           "content": [
             { "type": "text", "text": "Describe this food and estimate its calories." },
             { "type": "image_url", "image_url": "={{$binary.image.data}}" }
           ]
         }
       ]
       ```
   - Credentials: Connect OpenAI API credentials
   - Connect from Prepare Binary Image

6. **Add Google Sheets node (Log to Google Sheet):**
   - Type: Google Sheets
   - Parameters:
     - Range: "Sheet1!A:C"
     - Values: `={{ [[new Date().toISOString(), $json["text"]]] }}`
     - Sheet ID: Use your Google Sheet ID as a credential variable
     - Authentication: OAuth2
   - Credentials: Connect Google Sheets OAuth2 credentials
   - Connect from GPT-4 Vision Analyze

7. **Add Telegram node (Send GPT Result):**
   - Type: Telegram (Send Message)
   - Parameters:
     - Text: `={{$json["text"]}}`
     - Chat ID: `={{$json["message"]["chat"]["id"]}}`
   - Credentials: Use Telegram API credentials
   - Connect from GPT-4 Vision Analyze (parallel to Google Sheets node)

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                    |
|------------------------------------------------------------------------------------------------|---------------------------------------------------|
| Workflow demonstrates a practical use of GPT-4 Vision for image-based calorie estimation.       | n8n workflow example for AI-powered food tracking |
| Ensure Telegram bot has access to photo messages and permissions configured properly.           | Telegram Bot API documentation                     |
| Google Sheets OAuth2 credentials require enabling Sheets API and proper consent scope.          | https://developers.google.com/sheets/api/guides/authorizing |
| OpenAI GPT-4 Vision preview requires special API access and correct model naming.              | https://platform.openai.com/docs/models/gpt-4-vision |
| This workflow is inactive by default; activate after configuring all credentials and testing.  | n8n workflow management                             |

---

**Disclaimer:** The provided documentation exclusively describes an automated workflow created with n8n, respecting all applicable content policies and handling only legal, public data.