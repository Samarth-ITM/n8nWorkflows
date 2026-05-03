Automated Quiz Delivery from Google Sheets to Telegram with Status Tracking

https://n8nworkflows.xyz/workflows/automated-quiz-delivery-from-google-sheets-to-telegram-with-status-tracking-7520


# Automated Quiz Delivery from Google Sheets to Telegram with Status Tracking

### 1. Workflow Overview

This workflow automates the delivery of quiz questions stored in a Google Sheets document to a Telegram chat as polls, while tracking the status of each quiz question to avoid duplicates. It is ideal for educators, trainers, or community managers who wish to broadcast quizzes sequentially and monitor their completion status without manual intervention.

The logical structure is divided into these main blocks:

- **1.1 Input Reception:** Reads quiz data from Google Sheets.
- **1.2 Quiz Filtering:** Filters and selects the earliest pending quiz to be sent.
- **1.3 Conditional Routing:** Determines whether a pending quiz exists and branches accordingly.
- **1.4 Telegram Poll Dispatch:** Sends the quiz question as a Telegram poll.
- **1.5 Status Update:** Updates the quiz status in Google Sheets to mark it as sent.
- **1.6 Notification on Missing Quiz:** Alerts admins via Telegram if no pending quiz is found.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Reads all quiz entries from a specified Google Sheets document to retrieve quiz questions and their statuses.

**Nodes Involved:**  
- Read Quiz Data

**Node Details:**  

- **Read Quiz Data**  
  - Type: Google Sheets (Read operation)  
  - Configuration: Reads all rows from "Sheet1" of the Google Sheets document identified by a user-provided Document ID. No filtering applied at this stage.  
  - Key Expressions: Document ID parameter must be replaced with the actual Google Sheets ID.  
  - Inputs: None (trigger or start node implied)  
  - Outputs: Array of quiz rows with fields such as quiz_number, question, options (a-d), and status.  
  - Edge Cases: Errors may occur if the Google Sheets API credentials are invalid, the document ID is incorrect, or the sheet name does not exist. Rate limits or connectivity issues can also cause failures.  
  - Notes: The Google Sheet should have the following headers: quiz_number, question, option_a, option_b, option_c, option_d, status (sticky note reminder).

---

#### 1.2 Quiz Filtering

**Overview:**  
Filters the read quiz rows to identify only those marked as pending (status "🟨") and selects the earliest quiz by quiz_number to send next. If none are pending, it passes a marker item.

**Nodes Involved:**  
- Filter Pending Quiz

**Node Details:**  

- **Filter Pending Quiz**  
  - Type: Code (JavaScript)  
  - Configuration: Custom JavaScript filters items with status "🟨" and sorts them ascending by quiz_number, returning the first quiz. If none found, returns a dummy item with message "not exist".  
  - Key Expressions/Variables:  
    - `const pending = items.filter(i => i.json.status === '🟨')`  
    - Sorting by `quiz_number` cast to integer.  
  - Inputs: Output from "Read Quiz Data" node.  
  - Outputs: Either one pending quiz item or a marker item indicating no pending quiz.  
  - Edge Cases: If quiz_number is missing or not numeric, sorting may behave unexpectedly. If status is missing, item is excluded.  
  - Notes: Sticky note explains filtering logic clearly.

---

#### 1.3 Conditional Routing

**Overview:**  
Checks if the filtered quiz item has a valid status field. If yes, proceeds to send the quiz; if no, triggers a notification for missing quiz data.

**Nodes Involved:**  
- Check Quiz Exists

**Node Details:**  

- **Check Quiz Exists**  
  - Type: If Node  
  - Configuration: Condition tests if `status` field exists on the incoming JSON.  
  - Key Expressions: Uses `exists` operator on `{{ $json.status }}`  
  - Inputs: Output from "Filter Pending Quiz" node.  
  - Outputs:  
    - True branch: Quiz exists → send poll.  
    - False branch: No quiz → notify missing quiz.  
  - Edge Cases: If status is null, empty string, or missing, branch to notification. Expression failures if JSON structure unexpected.

---

#### 1.4 Telegram Poll Dispatch

**Overview:**  
Sends the selected quiz question as a poll to a Telegram chat using the Telegram Bot API.

**Nodes Involved:**  
- Send Telegram Poll

**Node Details:**  

- **Send Telegram Poll**  
  - Type: HTTP Request  
  - Configuration:  
    - Method: POST  
    - URL: Telegram Bot API endpoint `sendPoll` with Bot Token from environment variable `TELEGRAM_BOT_TOKEN`.  
    - Query Parameters:  
      - chat_id: from env `TELEGRAM_CHAT_ID`  
      - question: quiz question text from JSON  
      - options: array of options [option_a, option_b, option_c, option_d] formatted as JSON string  
  - Key Expressions:  
    - `{{ $env.TELEGRAM_BOT_TOKEN }}`, `{{ $env.TELEGRAM_CHAT_ID }}`  
    - `{{ $json.question }}`  
    - Options array with interpolation of option fields.  
  - Inputs: True branch from "Check Quiz Exists".  
  - Outputs: Response from Telegram API (poll sent confirmation).  
  - Edge Cases:  
    - Authentication errors if BOT_TOKEN invalid.  
    - Network timeouts.  
    - Telegram API limits.  
    - Missing or malformed question/options fields lead to API errors.  
  - Notes: Sticky note emphasizes the need to set environment variables.

---

#### 1.5 Status Update

**Overview:**  
Updates the quiz status in Google Sheets to "✅" to mark the quiz as sent and avoid resending.

**Nodes Involved:**  
- Prepare Status Update  
- Update Quiz Status1

**Node Details:**  

- **Prepare Status Update**  
  - Type: Set Node  
  - Configuration: Sets two fields:  
    - `status` to the string "✅"  
    - `quiz_number` copied from the original quiz row  
  - Key Expressions:  
    - `={{ $('Read Quiz Data').item.json.quiz_number }}` to retrieve quiz_number for matching during update.  
  - Inputs: Output from "Send Telegram Poll".  
  - Outputs: JSON object with updated status and quiz_number for update node.  
  - Edge Cases: If quiz_number missing, update may fail or update wrong row.  
  - Notes: Sticky note highlights purpose of marking quiz as sent.

- **Update Quiz Status1**  
  - Type: Google Sheets (Update operation)  
  - Configuration:  
    - Operation: Update row where quiz_number matches the provided quiz_number  
    - Sheet: "Sheet1" of same document ID as input  
    - Mapping Mode: Auto map input data to columns  
  - Inputs: Output from "Prepare Status Update".  
  - Outputs: Confirmation of update operation.  
  - Edge Cases:  
    - If quiz_number not unique or missing, might update multiple or zero rows.  
    - API errors due to invalid credentials or document ID.  
  - Notes: Sticky note clarifies update purpose.

---

#### 1.6 Notification on Missing Quiz

**Overview:**  
Sends a Telegram message to a specified admin chat if no pending quiz is found in the sheet, prompting refill action.

**Nodes Involved:**  
- Notify Missing Quiz (Telegram)

**Node Details:**  

- **Notify Missing Quiz (Telegram)**  
  - Type: Telegram Node (Send Message)  
  - Configuration:  
    - Text: "⚠️ No pending quiz found in Google Sheet. Please refill."  
    - chatId: from environment variable `TELEGRAM_NOTIFY_CHAT_ID`  
    - Additional: Attribution disabled  
  - Inputs: False branch from "Check Quiz Exists" node.  
  - Outputs: Confirmation of message sent.  
  - Edge Cases:  
    - Invalid or missing chat ID or bot token leads to failure.  
    - Network issues.  
  - Notes: Sticky note explains alert purpose and environment variable requirement.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                          | Input Node(s)           | Output Node(s)                | Sticky Note                                                                                  |
|--------------------------|---------------------|----------------------------------------|------------------------|------------------------------|----------------------------------------------------------------------------------------------|
| Read Quiz Data           | Google Sheets       | Read quiz rows from Google Sheets      | —                      | Filter Pending Quiz           | Create the sheet with this headers: quiz_number, question, option_a, option_b, option_c, option_d, status |
| Filter Pending Quiz      | Code                | Filter for earliest pending quiz       | Read Quiz Data          | Check Quiz Exists             | Filters rows marked 🟨 and selects the earliest quiz by number.                             |
| Check Quiz Exists        | If                  | Branch based on quiz existence          | Filter Pending Quiz     | Send Telegram Poll, Notify Missing Quiz (Telegram) | Branches depending on whether a pending quiz was found.                                     |
| Send Telegram Poll       | HTTP Request        | Send quiz as a Telegram poll            | Check Quiz Exists (true) | Prepare Status Update         | Sends the quiz as a poll to the group via Telegram Bot API.                                |
| Prepare Status Update    | Set                 | Prepare update data for quiz status     | Send Telegram Poll      | Update Quiz Status1           | Marks the quiz row as ✅ to prevent re-sending.                                             |
| Update Quiz Status1      | Google Sheets       | Update quiz status in Google Sheets     | Prepare Status Update   | —                            | Marks the quiz row as ✅ to prevent re-sending.                                             |
| Notify Missing Quiz (Telegram) | Telegram          | Notify admins if no pending quiz found | Check Quiz Exists (false) | —                            | Alerts admins if no pending quiz is available.                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheet**  
   - Create a Google Sheet named as desired.  
   - Add headers in "Sheet1": `quiz_number`, `question`, `option_a`, `option_b`, `option_c`, `option_d`, `status`.  
   - Populate rows with quiz data, marking pending quizzes with status "🟨".

2. **Add "Read Quiz Data" Node**  
   - Type: Google Sheets  
   - Operation: Read  
   - Sheet Name: `Sheet1`  
   - Document ID: Paste your Google Sheets document ID  
   - Credentials: Configure Google Sheets OAuth credentials with read access.

3. **Add "Filter Pending Quiz" Node**  
   - Type: Code (JavaScript)  
   - Input: Connect from "Read Quiz Data" output.  
   - Code:  
     ```js
     const items = $input.all();
     const pending = items.filter(i => i.json.status === '🟨');
     if (pending.length) {
       pending.sort((a,b) => parseInt(a.json.quiz_number) - parseInt(b.json.quiz_number));
       return [pending[0]];
     }
     return [{ json: { message: 'not exist' } }];
     ```
  
4. **Add "Check Quiz Exists" Node**  
   - Type: If  
   - Input: Connect from "Filter Pending Quiz" output.  
   - Condition: Check if `status` field exists in the JSON (`exists` operator on `{{ $json.status }}`).

5. **Add "Send Telegram Poll" Node**  
   - Type: HTTP Request  
   - Input: Connect from `true` output of "Check Quiz Exists".  
   - Method: POST  
   - URL: `https://api.telegram.org/bot{{ $env.TELEGRAM_BOT_TOKEN }}/sendPoll`  
   - Query Parameters:  
     - `chat_id` = `{{ $env.TELEGRAM_CHAT_ID }}`  
     - `question` = `{{ $json.question }}`  
     - `options` = JSON string array of options `[ "{{ $json.option_a }}", "{{ $json.option_b }}", "{{ $json.option_c }}", "{{ $json.option_d }}" ]`  
   - Environment Variables: Set `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` in your n8n environment.

6. **Add "Prepare Status Update" Node**  
   - Type: Set  
   - Input: Connect from "Send Telegram Poll" output.  
   - Set fields:  
     - `status` = `"✅"` (string)  
     - `quiz_number` = `={{ $('Read Quiz Data').item.json.quiz_number }}` (expression to get quiz_number from original data)

7. **Add "Update Quiz Status1" Node**  
   - Type: Google Sheets  
   - Operation: Update  
   - Sheet Name: `Sheet1`  
   - Document ID: Same as in "Read Quiz Data"  
   - Columns: Auto map input data  
   - Matching Columns: `quiz_number`  
   - Input: Connect from "Prepare Status Update" output.

8. **Add "Notify Missing Quiz (Telegram)" Node**  
   - Type: Telegram  
   - Input: Connect from `false` output of "Check Quiz Exists".  
   - Text: `"⚠️ No pending quiz found in Google Sheet. Please refill."`  
   - Chat ID: `{{ $env.TELEGRAM_NOTIFY_CHAT_ID }}` (set this environment variable)  
   - Additional Fields: Disable attribution.  
   - Environment Variables: Ensure `TELEGRAM_NOTIFY_CHAT_ID` is configured.

9. **Set Environment Variables**  
   - `TELEGRAM_BOT_TOKEN`: Your Telegram Bot API token.  
   - `TELEGRAM_CHAT_ID`: Chat ID where quizzes are sent.  
   - `TELEGRAM_NOTIFY_CHAT_ID`: Chat ID for admin notifications.

10. **Connect Nodes in Order**  
    - "Read Quiz Data" → "Filter Pending Quiz" → "Check Quiz Exists"  
    - "Check Quiz Exists" true → "Send Telegram Poll" → "Prepare Status Update" → "Update Quiz Status1"  
    - "Check Quiz Exists" false → "Notify Missing Quiz (Telegram)"

11. **Test and Deploy**  
    - Run the workflow manually or schedule as needed.  
    - Monitor logs for errors and correct credentials or permissions as required.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The Google Sheet must be structured with exact headers: quiz_number, question, option_a, option_b, option_c, option_d, status | Sticky Note in "Read Quiz Data" node                                                           |
| Telegram Bot API requires environment variables for security and ease of configuration: BOT_TOKEN, CHAT_ID | See Telegram Bot API documentation: https://core.telegram.org/bots/api#sendpoll                  |
| The status emoji "🟨" denotes pending quizzes; "✅" marks completed/sent quizzes to prevent duplicates     | Workflow logic and sticky notes                                                                |
| Notify Missing Quiz alerts admins to refill quizzes, ensuring continuous quiz delivery                    | Helps maintain workflow robustness                                                             |

---

This documentation fully describes the workflow structure, node-by-node configurations, and provides instructions for accurate reproduction and troubleshooting.