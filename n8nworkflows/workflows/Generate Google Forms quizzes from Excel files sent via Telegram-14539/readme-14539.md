Generate Google Forms quizzes from Excel files sent via Telegram

https://n8nworkflows.xyz/workflows/generate-google-forms-quizzes-from-excel-files-sent-via-telegram-14539


# Generate Google Forms quizzes from Excel files sent via Telegram

## 1. Workflow Overview

This workflow automates the creation of a Google Forms quiz from an Excel file sent through Telegram. A user sends a file to a Telegram bot, the workflow validates that the file is an Excel workbook, downloads and parses the content, creates a Google Form, enables quiz mode, then adds each question one by one with throttling to avoid API issues. At the end, it sends the generated Google Form link back to the user on Telegram.

### 1.1 Input Reception and File Validation
The workflow starts with a Telegram trigger. It checks whether the incoming file is an Excel document and immediately branches depending on validity: invalid files generate an error message, while valid files continue into processing.

### 1.2 File Download and Excel Parsing
Once the file is accepted, the workflow downloads it from Telegram, reads the spreadsheet contents, and normalizes the extracted rows into a question structure suitable for Google Forms.

### 1.3 Google Form Creation and Quiz Activation
After parsing the spreadsheet, the workflow creates a new Google Form via HTTP request and then updates the form settings so that it behaves as a quiz.

### 1.4 Question Preparation and Iterative Upload
The normalized question data is transformed into the payload expected by the Google Forms API. The workflow then loops through questions in batches, adding one question at a time.

### 1.5 Rate Limiting, Completion Check, and Final Notification
A wait node introduces a 2.5-second delay between question insertions. After each insertion, the workflow checks whether all questions have been processed; if yes, it sends the form link to the Telegram user, otherwise it loops back for the next question.

---

## 2. Block-by-Block Analysis

## 2.1 Input Reception and File Validation

**Overview:**  
This block receives a message from Telegram and determines whether the attached file is a supported Excel file. It acts as the workflow gatekeeper and prevents unsupported uploads from reaching the rest of the automation.

**Nodes Involved:**  
- Telegram Trigger – Receive File  
- Validate File Type (Excel Only)  
- If Not Excel → Send Error Message  
- Notify User – Invalid File Format  
- Notify User – Processing Started

### Node Details

#### Telegram Trigger – Receive File
- **Type and technical role:** `n8n-nodes-base.telegramTrigger`  
  Entry-point node that receives incoming Telegram bot updates through a webhook.
- **Configuration choices:**  
  The node is used as the workflow trigger. Since parameters are empty in the JSON, the expected behavior is a general Telegram update listener, likely intended for messages containing documents.
- **Key expressions or variables used:**  
  Not visible in the JSON, but downstream nodes will typically rely on fields such as:
  - chat ID
  - file ID
  - document metadata
  - message sender info
- **Input and output connections:**  
  - No input; this is the workflow start node.
  - Output → `Validate File Type (Excel Only)`
- **Version-specific requirements:**  
  `typeVersion: 1.2`
- **Edge cases or potential failure types:**  
  - Telegram bot token missing or invalid
  - Webhook not registered correctly
  - User sends text instead of a document
  - Telegram update payload shape differs depending on message type
- **Sub-workflow reference:** None

#### Validate File Type (Excel Only)
- **Type and technical role:** `n8n-nodes-base.code`  
  Custom validation logic that inspects the incoming Telegram file metadata to determine whether the file is an Excel format.
- **Configuration choices:**  
  Parameters are empty in exported JSON, which means the actual code body is not present here. Functionally, this node must evaluate file extension, MIME type, or both, and produce a boolean-like field used by the next IF node.
- **Key expressions or variables used:**  
  Likely reads from Telegram document fields such as:
  - filename
  - mime type
  - file ID
- **Input and output connections:**  
  - Input ← `Telegram Trigger – Receive File`
  - Output → `If Not Excel → Send Error Message`
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Missing `document` object in Telegram payload
  - Filename without extension
  - MIME type not provided
  - Code node runtime error due to undefined fields
- **Sub-workflow reference:** None

#### If Not Excel → Send Error Message
- **Type and technical role:** `n8n-nodes-base.if`  
  Conditional router that separates valid Excel uploads from invalid file submissions.
- **Configuration choices:**  
  Parameters are not exposed in the JSON, but behavior is clear from the outgoing connections:
  - True branch → valid Excel path
  - False branch → invalid file notification path
- **Key expressions or variables used:**  
  Likely checks a field created by the previous code node, such as `isExcel === true`.
- **Input and output connections:**  
  - Input ← `Validate File Type (Excel Only)`
  - Output 1 → `Download Excel from Telegram`
  - Output 1 → `Notify User – Processing Started`
  - Output 2 → `Notify User – Invalid File Format`
- **Version-specific requirements:**  
  `typeVersion: 2.2`
- **Edge cases or potential failure types:**  
  - Incorrect condition causes valid files to be rejected or invalid ones accepted
  - Missing validation field from previous code node
- **Sub-workflow reference:** None

#### Notify User – Invalid File Format
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends a Telegram message informing the user that only Excel files are accepted.
- **Configuration choices:**  
  Parameters are not present, but the node is clearly intended as an error-response message sender.
- **Key expressions or variables used:**  
  Likely uses:
  - incoming chat ID
  - static explanatory text
- **Input and output connections:**  
  - Input ← `If Not Excel → Send Error Message` (invalid-file branch)
  - No downstream output
- **Version-specific requirements:**  
  `typeVersion: 1.2`
- **Edge cases or potential failure types:**  
  - Telegram credential issues
  - Missing chat ID
  - User blocked the bot
- **Sub-workflow reference:** None

#### Notify User – Processing Started
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends an acknowledgment to the user that the Excel file was accepted and processing has started.
- **Configuration choices:**  
  Parameters omitted, but likely sends a simple status message to the same Telegram chat.
- **Key expressions or variables used:**  
  Likely uses the incoming chat ID and a static confirmation text.
- **Input and output connections:**  
  - Input ← `If Not Excel → Send Error Message` (valid-file branch)
  - No downstream output
- **Version-specific requirements:**  
  `typeVersion: 1.2`
- **Edge cases or potential failure types:**  
  Same as other Telegram send nodes.
- **Sub-workflow reference:** None

---

## 2.2 File Download and Excel Parsing

**Overview:**  
This block retrieves the accepted file from Telegram and converts spreadsheet rows into a normalized internal structure. Its output is the question dataset that feeds form creation and question generation.

**Nodes Involved:**  
- Download Excel from Telegram  
- Read Excel Data  
- Normalize Question Data

### Node Details

#### Download Excel from Telegram
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Uses Telegram API operations to fetch the binary file content corresponding to the document received in the trigger event.
- **Configuration choices:**  
  Parameters are not visible, but this node must be configured for file download rather than message sending. It likely uses the Telegram file ID from the trigger payload.
- **Key expressions or variables used:**  
  Likely references:
  - file ID from the incoming Telegram message
  - binary output property name
- **Input and output connections:**  
  - Input ← `If Not Excel → Send Error Message` (valid branch)
  - Output → `Read Excel Data`
- **Version-specific requirements:**  
  `typeVersion: 1.2`
- **Edge cases or potential failure types:**  
  - File ID missing
  - Telegram file no longer accessible
  - Large file download issues
  - Binary data not produced under expected property name
- **Sub-workflow reference:** None

#### Read Excel Data
- **Type and technical role:** `n8n-nodes-base.extractFromFile`  
  Extracts structured tabular data from the downloaded Excel file.
- **Configuration choices:**  
  Parameters are omitted, but this node is intended to read spreadsheet content from binary input and produce JSON rows.
- **Key expressions or variables used:**  
  Must reference the binary property coming from the Telegram download node.
- **Input and output connections:**  
  - Input ← `Download Excel from Telegram`
  - Output → `Normalize Question Data`
- **Version-specific requirements:**  
  `typeVersion: 1.1`
- **Edge cases or potential failure types:**  
  - Wrong binary property selected
  - Unsupported workbook format
  - Empty spreadsheet
  - Unexpected header row names
  - Multiple sheet handling ambiguity
- **Sub-workflow reference:** None

#### Normalize Question Data
- **Type and technical role:** `n8n-nodes-base.code`  
  Cleans and reshapes spreadsheet rows into a schema compatible with later Google Forms question generation.
- **Configuration choices:**  
  The code body is not present, but this node likely maps columns such as question text, answer options, and correct answer into a standard JSON array.
- **Key expressions or variables used:**  
  Likely transforms spreadsheet columns into fields such as:
  - question
  - options
  - correctAnswer
  - explanation or points if supported
- **Input and output connections:**  
  - Input ← `Read Excel Data`
  - Output → `Create Google Form`
- **Version-specific requirements:**  
  `typeVersion: 2`
- **Edge cases or potential failure types:**  
  - Missing expected columns
  - Blank rows
  - Inconsistent option count
  - Correct answer not matching listed options
  - Code node exceptions from malformed data
- **Sub-workflow reference:** None

---

## 2.3 Google Form Creation and Quiz Activation

**Overview:**  
This block creates the destination Google Form and updates its configuration so that it behaves like a quiz rather than a plain form. It establishes the form identifier needed by all subsequent question insertion requests.

**Nodes Involved:**  
- Create Google Form  
- Enable Quiz Mode

### Node Details

#### Create Google Form
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Google Forms API to create a new form.
- **Configuration choices:**  
  Parameters are not shown, but this node must include:
  - HTTP method likely `POST`
  - Google Forms API endpoint for form creation
  - OAuth2 or bearer token authentication
  - request body defining at least the form title
- **Key expressions or variables used:**  
  Likely derives the form name/title from normalized data, spreadsheet metadata, or static text.
- **Input and output connections:**  
  - Input ← `Normalize Question Data`
  - Output → `Enable Quiz Mode`
- **Version-specific requirements:**  
  `typeVersion: 4.3`
- **Edge cases or potential failure types:**  
  - Google API authentication failure
  - Missing Forms API enablement in Google Cloud
  - Invalid request body
  - Insufficient scopes
- **Sub-workflow reference:** None

#### Enable Quiz Mode
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends an update request to the Google Forms API to convert the created form into quiz mode.
- **Configuration choices:**  
  This is likely a `batchUpdate` style request on the created form ID, setting quiz-related options.
- **Key expressions or variables used:**  
  Likely uses:
  - form ID from `Create Google Form`
  - request body enabling quiz settings
- **Input and output connections:**  
  - Input ← `Create Google Form`
  - Output → `Prepare Question Payload`
- **Version-specific requirements:**  
  `typeVersion: 4.3`
- **Edge cases or potential failure types:**  
  - Form ID extraction fails
  - Incorrect batchUpdate payload
  - Google Forms API scope mismatch
- **Sub-workflow reference:** None

---

## 2.4 Question Preparation and Iterative Upload

**Overview:**  
This block transforms normalized question records into the exact payload needed by the Google Forms API, then processes them one by one through a loop. This design keeps insertion predictable and supports throttling between requests.

**Nodes Involved:**  
- Prepare Question Payload  
- Loop Questions (Batch Processing)  
- Add Question to Form

### Node Details

#### Prepare Question Payload
- **Type and technical role:** `n8n-nodes-base.set`  
  Prepares the per-question JSON structure consumed during the batch loop.
- **Configuration choices:**  
  The node likely maps form ID, question title, answer choices, and correct answer into a compact structure. Because it feeds a Split in Batches node, it probably emits one item per question.
- **Key expressions or variables used:**  
  Likely includes expressions referencing:
  - normalized question array
  - form ID from previous HTTP nodes
- **Input and output connections:**  
  - Input ← `Enable Quiz Mode`
  - Output → `Loop Questions (Batch Processing)`
- **Version-specific requirements:**  
  `typeVersion: 3.4`
- **Edge cases or potential failure types:**  
  - Set expressions refer to fields no longer present after prior nodes
  - Array expansion not handled correctly
  - Form ID lost across node transitions
- **Sub-workflow reference:** None

#### Loop Questions (Batch Processing)
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates over question items in controlled batches, almost certainly one at a time.
- **Configuration choices:**  
  Parameters are omitted, but the workflow pattern indicates standard loop behavior:
  - second output goes to per-item processing
  - return path re-enters the node for the next batch
- **Key expressions or variables used:**  
  Uses incoming question items prepared by the previous node.
- **Input and output connections:**  
  - Input ← `Prepare Question Payload`
  - Loop output → `Add Question to Form`
  - Completion path controlled by `Check All Questions Processed` feeding back into this node
- **Version-specific requirements:**  
  `typeVersion: 3`
- **Edge cases or potential failure types:**  
  - Batch size misconfigured
  - Empty item list causes no question creation
  - Loop never terminates if condition logic is wrong
- **Sub-workflow reference:** None

#### Add Question to Form
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the Google Forms API to append a question to the created quiz.
- **Configuration choices:**  
  This node likely sends a `batchUpdate` request containing one `createItem` instruction per loop iteration.
- **Key expressions or variables used:**  
  Likely uses:
  - form ID
  - question title
  - choice options
  - correct answer metadata
  - insertion index
- **Input and output connections:**  
  - Input ← `Loop Questions (Batch Processing)`
  - Output → `Rate Limit Delay (2.5s)`
- **Version-specific requirements:**  
  `typeVersion: 4.3`
- **Edge cases or potential failure types:**  
  - Invalid Google Forms question schema
  - Unsupported question type
  - Duplicate or missing options
  - API quota/rate limit issues
  - Authentication expiration mid-loop
- **Sub-workflow reference:** None

---

## 2.5 Rate Limiting, Completion Check, and Final Notification

**Overview:**  
This block slows down the loop to reduce API throttling risk, determines whether all questions have been processed, and sends the final Google Form link to the user once generation is complete.

**Nodes Involved:**  
- Rate Limit Delay (2.5s)  
- Check All Questions Processed  
- Send Google Form Link to User

### Node Details

#### Rate Limit Delay (2.5s)
- **Type and technical role:** `n8n-nodes-base.wait`  
  Introduces a pause between question insertions to avoid hitting Google API limits or update contention.
- **Configuration choices:**  
  The node name indicates a 2.5-second wait duration.
- **Key expressions or variables used:**  
  No expressions visible.
- **Input and output connections:**  
  - Input ← `Add Question to Form`
  - Output → `Check All Questions Processed`
- **Version-specific requirements:**  
  `typeVersion: 1.1`
- **Edge cases or potential failure types:**  
  - Wait node resumes unexpectedly late if queue load is high
  - Long workflows may accumulate execution duration
- **Sub-workflow reference:** None

#### Check All Questions Processed
- **Type and technical role:** `n8n-nodes-base.if`  
  Determines whether the loop has completed all question items or should continue with the next batch.
- **Configuration choices:**  
  Based on connections:
  - Output 1 → final user notification
  - Output 2 → loop continuation via `Loop Questions (Batch Processing)`
- **Key expressions or variables used:**  
  Likely tests Split in Batches completion state or remaining item count.
- **Input and output connections:**  
  - Input ← `Rate Limit Delay (2.5s)`
  - Output 1 → `Send Google Form Link to User`
  - Output 2 → `Loop Questions (Batch Processing)`
- **Version-specific requirements:**  
  `typeVersion: 2.2`
- **Edge cases or potential failure types:**  
  - Wrong condition can cause premature completion
  - Infinite loop if completion condition never becomes true
- **Sub-workflow reference:** None

#### Send Google Form Link to User
- **Type and technical role:** `n8n-nodes-base.telegram`  
  Sends the generated Google Form URL back to the Telegram user.
- **Configuration choices:**  
  Parameters are hidden, but this node likely includes a message text with the form edit or responder link.
- **Key expressions or variables used:**  
  Likely references:
  - Telegram chat ID from the trigger
  - form URL from `Create Google Form`
- **Input and output connections:**  
  - Input ← `Check All Questions Processed`
  - No downstream output
- **Version-specific requirements:**  
  `typeVersion: 1.2`
- **Edge cases or potential failure types:**  
  - Form URL field not preserved through the loop
  - Telegram API message failure
- **Sub-workflow reference:** None

---

## 2.6 Sticky Notes and Workspace Annotation

**Overview:**  
These nodes are canvas annotations only. They do not affect execution but may visually group sections of the workflow.

**Nodes Involved:**  
- Sticky Note  
- Sticky Note1  
- Sticky Note2  
- Sticky Note3  
- Sticky Note4  
- Sticky Note5

### Node Details

All six are of type `n8n-nodes-base.stickyNote`, version `1`, with empty content in the provided JSON.

- **Type and technical role:** Visual documentation nodes.
- **Configuration choices:** All sticky notes have empty content.
- **Key expressions or variables used:** None
- **Input and output connections:** None
- **Version-specific requirements:** `typeVersion: 1`
- **Edge cases or potential failure types:** None operational
- **Sub-workflow reference:** None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Telegram Trigger – Receive File | telegramTrigger | Receives incoming Telegram document/message updates |  | Validate File Type (Excel Only) |  |
| Validate File Type (Excel Only) | code | Validates whether the uploaded file is an Excel file | Telegram Trigger – Receive File | If Not Excel → Send Error Message |  |
| If Not Excel → Send Error Message | if | Routes valid Excel files to processing and invalid files to error handling | Validate File Type (Excel Only) | Download Excel from Telegram; Notify User – Processing Started; Notify User – Invalid File Format |  |
| Notify User – Invalid File Format | telegram | Sends error response for unsupported file formats | If Not Excel → Send Error Message |  |  |
| Notify User – Processing Started | telegram | Sends acknowledgment that processing has started | If Not Excel → Send Error Message |  |  |
| Download Excel from Telegram | telegram | Downloads the file binary from Telegram | If Not Excel → Send Error Message | Read Excel Data |  |
| Read Excel Data | extractFromFile | Parses Excel workbook into structured rows | Download Excel from Telegram | Normalize Question Data |  |
| Normalize Question Data | code | Reshapes spreadsheet rows into question objects | Read Excel Data | Create Google Form |  |
| Create Google Form | httpRequest | Creates a new Google Form through API | Normalize Question Data | Enable Quiz Mode |  |
| Enable Quiz Mode | httpRequest | Converts the created form into a quiz | Create Google Form | Prepare Question Payload |  |
| Prepare Question Payload | set | Maps normalized questions into Google Forms request payload items | Enable Quiz Mode | Loop Questions (Batch Processing) |  |
| Loop Questions (Batch Processing) | splitInBatches | Iterates through questions one batch at a time | Prepare Question Payload; Check All Questions Processed | Add Question to Form |  |
| Add Question to Form | httpRequest | Appends a question to the Google Form | Loop Questions (Batch Processing) | Rate Limit Delay (2.5s) |  |
| Rate Limit Delay (2.5s) | wait | Pauses between question insertions | Add Question to Form | Check All Questions Processed |  |
| Check All Questions Processed | if | Determines whether to continue looping or finish | Rate Limit Delay (2.5s) | Send Google Form Link to User; Loop Questions (Batch Processing) |  |
| Send Google Form Link to User | telegram | Sends the generated Google Form link to the user | Check All Questions Processed |  |  |
| Sticky Note | stickyNote | Canvas annotation |  |  |  |
| Sticky Note1 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note2 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note3 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note4 | stickyNote | Canvas annotation |  |  |  |
| Sticky Note5 | stickyNote | Canvas annotation |  |  |  |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**  
   Set the workflow name to something like:  
   `Generate Google Form Quiz from Excel via Telegram`.

2. **Add a Telegram Trigger node**
   - Node type: `Telegram Trigger`
   - Name it: `Telegram Trigger – Receive File`
   - Configure Telegram bot credentials.
   - Set it to listen for incoming updates from the bot.
   - Ensure the bot is allowed to receive documents.

3. **Add a Code node for file validation**
   - Node type: `Code`
   - Name it: `Validate File Type (Excel Only)`
   - Connect it after the Telegram trigger.
   - In the code, inspect the Telegram document metadata.
   - Check whether the filename or MIME type matches Excel formats such as:
     - `.xlsx`
     - `.xls`
   - Return a field such as `isExcel`.

4. **Add an IF node for routing**
   - Node type: `If`
   - Name it: `If Not Excel → Send Error Message`
   - Connect it after the validation code node.
   - Configure the condition to test whether `isExcel` is true.

5. **Add a Telegram node for invalid file errors**
   - Node type: `Telegram`
   - Name it: `Notify User – Invalid File Format`
   - Connect it to the false branch of the IF node.
   - Configure it to send a message to the original chat ID.
   - Example message:  
     `Please send a valid Excel file (.xlsx or .xls) containing quiz questions.`

6. **Add a Telegram node for start notification**
   - Node type: `Telegram`
   - Name it: `Notify User – Processing Started`
   - Connect it to the true branch of the IF node.
   - Configure it to send a message to the same chat ID.
   - Example message:  
     `Your Excel file was received. Quiz generation has started.`

7. **Add a Telegram node to download the file**
   - Node type: `Telegram`
   - Name it: `Download Excel from Telegram`
   - Also connect it to the true branch of the IF node.
   - Configure it to download the document using the Telegram `file_id` from the trigger payload.
   - Ensure the output is binary data.

8. **Add an Extract From File node**
   - Node type: `Extract From File`
   - Name it: `Read Excel Data`
   - Connect it after `Download Excel from Telegram`.
   - Configure it to read the binary file as Excel.
   - Select the correct binary property name from the download node.
   - If needed, specify the worksheet to read or use the default first sheet.

9. **Add a Code node to normalize spreadsheet rows**
   - Node type: `Code`
   - Name it: `Normalize Question Data`
   - Connect it after `Read Excel Data`.
   - Convert spreadsheet rows into a normalized JSON structure.
   - Expected fields per question should include at least:
     - question text
     - answer choices
     - correct answer
   - If your source format includes more data, you can also support:
     - points
     - explanation
     - question type

10. **Prepare Google API access**
    - Since the workflow uses raw HTTP nodes, configure Google OAuth2 credentials usable with the Google Forms API.
    - In Google Cloud:
      - enable the Google Forms API
      - create OAuth credentials
      - authorize the scopes required for form creation and updates
    - Attach these credentials to the HTTP Request nodes.

11. **Add an HTTP Request node to create the form**
    - Node type: `HTTP Request`
    - Name it: `Create Google Form`
    - Connect it after `Normalize Question Data`.
    - Configure:
      - Method: `POST`
      - URL: Google Forms API endpoint for form creation
      - Authentication: Google OAuth2 or bearer token
      - Body: include form info such as title/document title
    - Preserve the returned form ID and form URL in the output.

12. **Add an HTTP Request node to enable quiz mode**
    - Node type: `HTTP Request`
    - Name it: `Enable Quiz Mode`
    - Connect it after `Create Google Form`.
    - Configure it to call the form update endpoint, usually a `batchUpdate` request.
    - Use the form ID returned by the previous node.
    - In the request body, enable quiz settings.

13. **Add a Set node to prepare question payload items**
    - Node type: `Set`
    - Name it: `Prepare Question Payload`
    - Connect it after `Enable Quiz Mode`.
    - Map each normalized question into the structure expected by Google Forms.
    - Include:
      - form ID
      - question title
      - options list
      - correct answer
      - insertion index if needed
    - Make sure the form metadata remains available for the final notification.

14. **Add a Split In Batches node**
    - Node type: `Split In Batches`
    - Name it: `Loop Questions (Batch Processing)`
    - Connect it after `Prepare Question Payload`.
    - Set batch size to `1` so questions are added one at a time.

15. **Add an HTTP Request node to insert one question**
    - Node type: `HTTP Request`
    - Name it: `Add Question to Form`
    - Connect it to the loop output of the Split In Batches node.
    - Configure:
      - Method: likely `POST`
      - URL: Google Forms `batchUpdate` endpoint for the specific form
      - Authentication: same Google credentials
      - Body: a single question insertion request using the current loop item
    - Build the body using expressions from the current question item.

16. **Add a Wait node**
    - Node type: `Wait`
    - Name it: `Rate Limit Delay (2.5s)`
    - Connect it after `Add Question to Form`.
    - Configure a wait duration of 2.5 seconds.

17. **Add an IF node to detect completion**
    - Node type: `If`
    - Name it: `Check All Questions Processed`
    - Connect it after the wait node.
    - Configure it so that:
      - if all items have been processed, go to final notification
      - otherwise, continue looping
    - This is usually based on Split In Batches loop state.

18. **Create the loop-back connection**
    - Connect the non-complete branch of `Check All Questions Processed` back into `Loop Questions (Batch Processing)`.
    - This is what advances the next batch/item.

19. **Add a Telegram node to send the final form link**
    - Node type: `Telegram`
    - Name it: `Send Google Form Link to User`
    - Connect it to the complete branch of `Check All Questions Processed`.
    - Configure it to send a message containing:
      - the form URL
      - optionally the edit URL if relevant
    - Use the original chat ID from the trigger node.

20. **Optional: add sticky notes**
    - Add `Sticky Note` nodes around each block for visual clarity:
      - input/validation
      - file extraction
      - form creation
      - question loop
      - final notification

21. **Test with a sample Excel file**
    - Send a known-good `.xlsx` file to the Telegram bot.
    - Confirm:
      - the user receives the “processing started” message
      - the file is parsed correctly
      - a Google Form is created
      - quiz mode is enabled
      - all questions are added
      - the final form link is sent back

22. **Validate edge cases**
    - Send a `.pdf` or `.txt` file to verify invalid-file handling.
    - Test an Excel file with:
      - empty rows
      - malformed columns
      - missing correct answers
    - Confirm the Code nodes handle these cases safely.

### Expected Input/Output Design for the Excel File
To make this workflow reliable, define a consistent spreadsheet schema. A practical example is:

| Question | Option A | Option B | Option C | Option D | Correct Answer |
|---|---|---|---|---|---|

Where:
- `Question` contains the question text
- `Option A` to `Option D` contain answer choices
- `Correct Answer` contains either:
  - the exact option text, or
  - the option label such as `A`, `B`, `C`, `D`

### Required Credentials
- **Telegram Bot API credentials**
  - Used by:
    - Telegram Trigger – Receive File
    - Notify User – Invalid File Format
    - Notify User – Processing Started
    - Download Excel from Telegram
    - Send Google Form Link to User
- **Google OAuth2 credentials**
  - Used by:
    - Create Google Form
    - Enable Quiz Mode
    - Add Question to Form

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| No non-empty sticky note content was present in the provided workflow JSON. | Workspace annotations are empty in this export. |
| The workflow depends on the Google Forms API even though it uses generic HTTP Request nodes rather than a dedicated Google Forms node. | Enable the API in Google Cloud before testing. |
| The logic strongly depends on spreadsheet column consistency. Standardize the Excel schema before production use. | Applies to `Read Excel Data` and `Normalize Question Data`. |
| The loop includes a 2.5-second delay to reduce API throttling risk. | Applies to the question insertion sequence. |

If you want, I can also produce a second version of this document with:
1. inferred example expressions for each node, and  
2. a proposed Excel column schema plus sample Code node scripts.