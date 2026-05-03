Daily Govt Exam Quiz by Gopal Debnath (mrtechyguru)

https://n8nworkflows.xyz/workflows/daily-govt-exam-quiz-by-gopal-debnath--mrtechyguru--5284


# Daily Govt Exam Quiz by Gopal Debnath (mrtechyguru)

### 1. Workflow Overview

This workflow, titled **Daily Govt Exam Quiz Delivery**, automates the daily distribution of a government exam quiz question to users via multiple communication channels: email, Telegram, and SMS (Twilio). It is designed for quiz administrators or educators who want to engage their audience with daily quiz questions extracted from a Google Sheet.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Trigger:** Initiates the workflow every day at a fixed time.
- **1.2 Data Retrieval:** Fetches quiz questions stored in a Google Sheet.
- **1.3 Quiz Formatting:** Randomly selects and formats a quiz question with options, answer, and explanation.
- **1.4 Multi-Channel Delivery:** Sends the formatted quiz via Email, Telegram, and SMS.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger

- **Overview:**  
  Triggers the workflow execution automatically every day at 6:00 AM.

- **Nodes Involved:**  
  - Daily Trigger

- **Node Details:**  
  - **Daily Trigger**  
    - *Type:* Cron Trigger  
    - *Role:* Starts the workflow daily at a specified time  
    - *Configuration:* Cron expression set to `0 6 * * *` (6 AM daily)  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Connects to "Fetch Question" node  
    - *Edge Cases:*  
      - Cron expression misconfiguration could lead to incorrect trigger times.  
      - Workflow will not run if n8n instance is down at trigger time.

#### 2.2 Data Retrieval

- **Overview:**  
  Retrieves quiz questions from a Google Sheet containing columns for question, multiple options, correct answer, and explanation.

- **Nodes Involved:**  
  - Fetch Question

- **Node Details:**  
  - **Fetch Question**  
    - *Type:* Google Sheets node  
    - *Role:* Reads quiz questions data from a specified Google Sheet range  
    - *Configuration:*  
      - Spreadsheet ID set to `"YOUR_GOOGLE_SHEET_ID"` (to be replaced with actual sheet ID)  
      - Range: `Sheet1!A2:F` (assumes data starts from row 2, columns A to F)  
      - Value Render Mode: Formatted values (human-readable)  
    - *Inputs:* From "Daily Trigger" node  
    - *Outputs:* Connects to "Format Quiz" node  
    - *Credentials:* Google API OAuth credentials required  
    - *Edge Cases:*  
      - Invalid or missing Google Sheet ID will cause failure.  
      - Insufficient permissions or expired OAuth token leads to authentication errors.  
      - Empty or malformed sheet data may cause downstream errors.  
      - Large data sets may lead to timeouts or performance delays.

#### 2.3 Quiz Formatting

- **Overview:**  
  Randomly selects one quiz question from the fetched data, formats the question with options, correct answer, and explanation into a structured text message suitable for sending.

- **Nodes Involved:**  
  - Format Quiz

- **Node Details:**  
  - **Format Quiz**  
    - *Type:* Function node (JavaScript code execution)  
    - *Role:* Processes the array of questions and formats one into a message  
    - *Configuration Highlights:*  
      - Randomly selects one item from the input array `items`.  
      - Constructs a message including question, options labeled A-D, the correct answer, and explanation.  
      - Creates a `fullText` field with Markdown-style formatting for rich message presentation.  
    - *Key Variables:*  
      - `items`: input array of questions  
      - `data`: chosen question object  
      - Outputs an object with keys: `question`, `options`, `correct`, `explanation`, `fullText`  
    - *Inputs:* From "Fetch Question" node  
    - *Outputs:* To "Send Email", "Send to Telegram", "Send via Twilio" nodes  
    - *Edge Cases:*  
      - Input array empty or undefined causes runtime error.  
      - Missing fields in data (e.g., no explanation) handled by default text.  
      - Special characters in question/options may need escaping if channels do not support Markdown fully.

#### 2.4 Multi-Channel Delivery

- **Overview:**  
  Sends the formatted quiz question text via Email, Telegram, and SMS to predefined recipients.

- **Nodes Involved:**  
  - Send Email  
  - Send to Telegram  
  - Send via Twilio

- **Node Details:**  

  - **Send Email**  
    - *Type:* Email Send node  
    - *Role:* Sends an email with the quiz question  
    - *Configuration:*  
      - Subject: "Your Daily Govt Exam Quiz"  
      - To: `candidate@example.com` (replace with actual recipient)  
      - From: `yourname@example.com` (replace with sender's email)  
      - Email body text set via expression `{{$json["fullText"]}}` (formatted quiz)  
    - *Credentials:* SMTP credentials required  
    - *Inputs:* From "Format Quiz" node  
    - *Outputs:* None (end node)  
    - *Edge Cases:*  
      - SMTP authentication failure or invalid credentials.  
      - Incorrect email addresses cause delivery failure.  
      - Email body formatting depends on SMTP server capabilities.

  - **Send to Telegram**  
    - *Type:* Telegram node  
    - *Role:* Sends the quiz text message to a Telegram chat/group  
    - *Configuration:*  
      - Chat ID: `"YOUR_TELEGRAM_CHAT_ID"` (to be replaced)  
      - Message text via expression `{{$json["fullText"]}}`  
    - *Credentials:* Telegram Bot API credentials required  
    - *Inputs:* From "Format Quiz" node  
    - *Outputs:* None  
    - *Edge Cases:*  
      - Invalid chat ID or revoked bot permissions cause failure.  
      - Telegram message length limits (max 4096 chars).  
      - Network or API rate limits.

  - **Send via Twilio**  
    - *Type:* Twilio node  
    - *Role:* Sends the quiz text as SMS to a phone number  
    - *Configuration:*  
      - To: `+919999999999` (recipient phone number, replace as needed)  
      - From: `+1234567890` (Twilio verified number)  
      - Message: `{{$json["fullText"]}}`  
    - *Credentials:* Twilio API credentials required  
    - *Inputs:* From "Format Quiz" node  
    - *Outputs:* None  
    - *Edge Cases:*  
      - Invalid phone numbers or unverified sender numbers cause failure.  
      - SMS length limits (usually 160 chars per segment); long messages may be split or truncated.  
      - Twilio API limits and network issues.

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role               | Input Node(s)    | Output Node(s)                         | Sticky Note                                                   |
|-------------------|---------------------|------------------------------|------------------|--------------------------------------|---------------------------------------------------------------|
| Daily Trigger     | Cron Trigger        | Initiates workflow daily at 6 AM | None             | Fetch Question                       |                                                               |
| Fetch Question    | Google Sheets       | Retrieves quiz questions from Google Sheet | Daily Trigger    | Format Quiz                         | Requires Google API credentials and correct Sheet ID.         |
| Format Quiz       | Function            | Selects and formats quiz question for delivery | Fetch Question   | Send Email, Send to Telegram, Send via Twilio | Handles missing explanation with default text.                |
| Send Email        | Email Send          | Sends quiz question by email | Format Quiz       | None                                | Requires SMTP credentials; replace recipient and sender emails. |
| Send to Telegram  | Telegram            | Sends quiz question via Telegram bot | Format Quiz       | None                                | Requires Telegram Bot API credentials and valid chat ID.      |
| Send via Twilio   | Twilio              | Sends quiz question via SMS  | Format Quiz       | None                                | Requires Twilio credentials; phone numbers must be valid.     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it "Daily Govt Exam Quiz Delivery".

2. **Add a Cron Trigger node**:
   - Name: `Daily Trigger`
   - Type: Cron
   - Set Cron Expression to `0 6 * * *` (to trigger every day at 6:00 AM)
   - No credentials required
   - Save and connect its output to the next node.

3. **Add a Google Sheets node**:
   - Name: `Fetch Question`
   - Operation: Read Rows
   - Sheet ID: Replace `"YOUR_GOOGLE_SHEET_ID"` with your actual Google Sheet ID containing quiz questions
   - Range: Set to `Sheet1!A2:F` assuming data starts at row 2 and columns A-F are used (question, options A-D, correct answer, explanation)
   - Value Render Mode: Formatted Value
   - Credentials: Select or create Google API OAuth2 credentials with access to your Sheet
   - Connect input from `Daily Trigger` node

4. **Add a Function node**:
   - Name: `Format Quiz`
   - Paste the following JavaScript code into the Function Code field:
     ```javascript
     const data = items[Math.floor(Math.random() * items.length)].json;
     return [{
       json: {
         question: data.question,
         options: `A. ${data.optionA}\nB. ${data.optionB}\nC. ${data.optionC}\nD. ${data.optionD}`,
         correct: data.correctAnswer,
         explanation: data.explanation || "No explanation provided.",
         fullText: `🧠 *Daily Quiz Question*\n\n*Q:* ${data.question}\n\n${data.optionA ? "A. " + data.optionA : ""}\n${data.optionB ? "B. " + data.optionB : ""}\n${data.optionC ? "C. " + data.optionC : ""}\n${data.optionD ? "D. " + data.optionD : ""}\n\n*Answer:* ${data.correctAnswer}\n📘 Explanation: ${data.explanation || "N/A"}`
       }
     }];
     ```
   - Connect input from `Fetch Question` node.

5. **Add an Email Send node**:
   - Name: `Send Email`
   - Set "To Email" to the recipient's email address (e.g., `candidate@example.com`)
   - Set "From Email" to your sender email (e.g., `yourname@example.com`)
   - Subject: `Your Daily Govt Exam Quiz`
   - Text: Use expression editor to set: `{{$json["fullText"]}}`
   - Credentials: Create or select SMTP credentials for your email server
   - Connect input from `Format Quiz` node

6. **Add a Telegram node**:
   - Name: `Send to Telegram`
   - Chat ID: Replace `"YOUR_TELEGRAM_CHAT_ID"` with the chat or channel ID where you want to send the quiz
   - Text: Use expression `{{$json["fullText"]}}`
   - Credentials: Create or select a Telegram Bot API credential
   - Connect input from `Format Quiz` node

7. **Add a Twilio node**:
   - Name: `Send via Twilio`
   - To: Set recipient phone number in E.164 format (e.g., `+919999999999`)
   - From: Your Twilio phone number (must be verified) (e.g., `+1234567890`)
   - Message: Use expression `{{$json["fullText"]}}`
   - Credentials: Create or select Twilio API credentials
   - Connect input from `Format Quiz` node

8. **Test the workflow** by manually triggering the `Daily Trigger` node or waiting for the scheduled time.

9. **Activate the workflow** once all tests pass.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                              |
|-----------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Replace all placeholder values (`YOUR_GOOGLE_SHEET_ID`, `candidate@example.com`, `YOUR_TELEGRAM_CHAT_ID`, phone numbers) with real data before activation. | Important for correct functioning and delivery.              |
| Google Sheets data should be structured: Question, Option A, Option B, Option C, Option D, Correct Answer, Explanation starting from row 2. | Data preparation for the quiz source.                         |
| Ensure all credentials (Google API, SMTP, Telegram Bot, Twilio) are valid, active, and have sufficient permissions. | Authentication and authorization critical for delivery.      |
| SMS messages may be truncated if exceeding character limits; consider shortening questions or explanations if needed. | SMS channel constraints.                                      |
| Telegram supports Markdown formatting used in `fullText`; test message appearance in your chat before full deployment. | Message formatting considerations.                            |
| For SMTP, consider using secure connections (TLS/SSL) and verify sender email reputation to avoid spam filters. | Email deliverability best practices.                          |
| Twilio phone number must be verified and active in your Twilio account to send SMS.                       | Twilio account setup requirement.                             |
| This workflow can be adapted to other quiz topics or schedules by changing the Google Sheet data and cron timing. | Flexibility and customization note.                           |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.