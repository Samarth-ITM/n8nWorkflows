Manage Construction Projects with Tasks, Photo Reports, Telegram & Google Sheets

https://n8nworkflows.xyz/workflows/manage-construction-projects-with-tasks--photo-reports--telegram---google-sheets-6675


# Manage Construction Projects with Tasks, Photo Reports, Telegram & Google Sheets

---

### 1. Workflow Overview

This n8n workflow automates the management of construction projects by integrating task tracking, photo report reminders, Telegram messaging, and Google Sheets data storage. It primarily serves project managers and on-site executors by automating reminders, status updates, and report submissions, ensuring timely communication and recordkeeping.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Reminders for Tasks and Photo Reports:** Periodic checks to identify pending tasks and photo reports, forming and sending reminders via Telegram, and updating Google Sheets with reminder statuses.
  
- **1.2 Incoming Telegram Message Handling:** A central webhook processes all incoming Telegram messages, including commands, photo report submissions, text task reports, and geolocation data.

- **1.3 Registration and User Management:** Handles new user registration and verification against the Users Google Sheet, sending welcome messages accordingly.

- **1.4 Task and Photo Report Processing:** Parses incoming reports (text and photo), validates task ownership, uploads photos to Google Drive, updates Google Sheets, and sends confirmations.

- **1.5 Read and Received Confirmations:** Processes user confirmations of reading reminders and receiving tasks, updating statuses on Google Sheets and sending acknowledgments.

- **1.6 Geolocation Processing:** Extracts GPS coordinates sent in reply to reminders, updates photo reports with location data, and confirms receipt.

- **1.7 Error Handling and User Guidance:** Manages errors such as missing replies, file size limits, or invalid task IDs, providing users with helpful messages.

- **1.8 Data Grouping and Updates:** Groups multiple photo files per task, merges with existing records, and updates or creates entries in Google Sheets accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Reminders for Tasks and Photo Reports

**Overview:**  
This block runs every minute to check the "Tasks" and "Photo Reports" Google Sheets for pending reminders and sends personalized Telegram messages to executors. It also updates the reminder status in Google Sheets after sending.

**Nodes Involved:**  
- Check every minute - Tasks (Cron)  
- Get tasks (Google Sheets)  
- Find and form messages - Tasks (Code)  
- Are there messages to send? - Tasks (If)  
- Send to Telegram - Tasks (Telegram)  
- Prepare updates - Tasks (Code)  
- Are there updates? - Tasks (If)  
- Update status in table - Tasks (Google Sheets)  
- No messages - Tasks (NoOp)  
- No updates - Tasks (NoOp)  
- Check every minute - Photo (Cron)  
- Get photo reports data (Google Sheets)  
- Find and form messages - Photo (Code)  
- Are there messages to send? (If)  
- Send to Telegram - Photo (Telegram)  
- Prepare Updates (Code)  
- Are there updates? (If)  
- Update status in table - Photo (Google Sheets)  
- No messages (NoOp)  
- No updates (NoOp)  

**Node Details:**

- **Check every minute - Tasks & Photo**  
  - Type: Cron  
  - Role: Triggers the workflow every minute for both tasks and photo reports.  
  - Configuration: Runs "everyMinute" schedule.  
  - Failure: Cron misconfiguration or workflow not active will stop triggers.

- **Get tasks & Get photo reports data**  
  - Type: Google Sheets  
  - Role: Reads the "Tasks" and "Photo Reports" sheets respectively.  
  - Configuration: Reads entire sheets by name, requires OAuth2 credentials.  
  - Failure: Auth errors, API limits, or invalid document/sheet IDs.

- **Find and form messages - Tasks & Photo**  
  - Type: Code  
  - Role: Parses rows, compares dates and times against current time, generates messages and tracks executors to notify with pending reminders.  
  - Uses extensive date/time parsing and validation, including Excel date/time formats.  
  - Failure: Parsing errors of dates/times, missing critical fields, timezone mismatches.

- **Are there messages to send? - Tasks & Photo**  
  - Type: If  
  - Role: Checks if any messages are generated to send.  
  - Output: Branches to sending or NoOp nodes.

- **Send to Telegram - Tasks & Photo**  
  - Type: Telegram  
  - Role: Sends messages to executors' Telegram chats.  
  - Configuration: Uses Telegram API credentials, sends HTML-formatted messages.  
  - Failure: Telegram API errors, invalid chat IDs, rate limits.

- **Prepare updates - Tasks & Prepare Updates**  
  - Type: Code  
  - Role: Prepares the data to update reminder status columns ("Reminder Sent", "Reminder Count", "Reminder Sent Date") in Google Sheets.  
  - Failure: Incorrect message data format, missing row indexes.

- **Are there updates? - Tasks & Are there updates?**  
  - Type: If  
  - Role: Checks if there are any updates to apply.  

- **Update status in table - Tasks & Update status in table - Photo**  
  - Type: Google Sheets  
  - Role: Updates reminder status columns in corresponding sheets.  
  - Configuration: Uses row_number matching for updates.  
  - Failure: Sheet access issues, concurrency conflicts.

- **No messages & No messages - Tasks, No updates & No updates - Tasks**  
  - Type: NoOp  
  - Role: Terminal nodes when no messages or updates exist.

---

#### 1.2 Incoming Telegram Message Handling

**Overview:**  
Central webhook node receives all Telegram updates (messages, photos, documents) and routes to appropriate processing nodes depending on the message content and user actions.

**Nodes Involved:**  
- Telegram Webhook - Receiving messages (Telegram Trigger)  
- Process incoming message1 (Code)  

**Node Details:**

- **Telegram Webhook - Receiving messages**  
  - Type: Telegram Trigger  
  - Role: Entry point for all Telegram user messages and actions.  
  - Configuration: Listens to "message", "photo", and "document" updates.  
  - Failure: Telegram webhook misconfiguration, connectivity issues.

- **Process incoming message1**  
  - Type: Code  
  - Role: Parses incoming message content and determines action to take, including command parsing (/start, /help, /status, /report, /read, /received), photo and geolocation processing, text report parsing, and registration.  
  - Contains detailed parsing logic for multiple message types and reply message context.  
  - Failure: Complex parsing logic may fail on unexpected message formats; missing reply_to_message leads to error messages.

---

#### 1.3 Registration and User Management

**Overview:**  
Handles verifying if a Telegram user is registered, processing new registrations, requesting additional user data, and confirming registration status.

**Nodes Involved:**  
- Check register_user1 (If)  
- Read users sheet1 (Google Sheets)  
- Check registration1 (Code)  
- Check new user1 (If)  
- Request registration data1 (Code)  
- Check existing user1 (If)  
- Generate welcome1 (Code)  
- Confirm registration (Code)  
- Prepare data for recording (Code)  
- Record new executor (Google Sheets)  
- Send message (Telegram)  
- Send message2 (Telegram)  

**Node Details:**

- **Check register_user1**  
  - Checks if action is 'register_user' to trigger registration logic.

- **Read users sheet1**  
  - Reads Users sheet filtering on Telegram ID.

- **Check registration1**  
  - Checks if user exists in Users sheet, branches accordingly.

- **Check new user1**  
  - Branches whether user is new or existing.

- **Request registration data1**  
  - Sends message requesting user to provide name and company.

- **Check existing user1**  
  - Checks if action is to send welcome to existing user.

- **Generate welcome1**  
  - Composes welcome messages for new or returning users.

- **Confirm registration**  
  - Creates confirmation message after registration data is saved.

- **Prepare data for recording**  
  - Adds registration date and time before saving.

- **Record new executor**  
  - Appends new user data to Users Google Sheet.

- **Send message & Send message2**  
  - Telegram nodes sending messages to users.

**Failures:**  
- Google Sheets access or write failure.  
- Missing or malformed registration data.  
- Telegram message delivery failure.

---

#### 1.4 Task and Photo Report Processing

**Overview:**  
Processes incoming photos and text messages related to tasks and photo reports, validates ownership, uploads files to Google Drive, updates Google Sheets entries, and sends confirmation messages back to executors.

**Nodes Involved:**  
- Check process_photo3 (If)  
- Extract photo data4 (Code)  
- Check error2 (If)  
- Photo error message4 (Telegram)  
- Get and download file4 (Telegram)  
- Check file presence4 (If)  
- Upload to Google Drive4 (Google Drive)  
- Handle retrieval error4 (Code)  
- Merge file streams2 (Merge)  
- Group files by task (Code)  
- Check existing task1 (Google Sheets)  
- Does task exist?1 (If)  
- Prepare data for adding2 (Code)  
- Prepare data for updating2 (Code)  
- Add to reports table1 (Google Sheets)  
- Update row in table2 (Google Sheets)  
- Merge lookup and grouped files (Merge)  
- Prepare confirmation (Code)  
- Confirm photo receipt4 (Telegram)  
- Check Task Owner1 (If)  
- Get Tasks for Check1 (Google Sheets)  
- Check ID Match1 (Code)  
- Check Process Report1 (If)  
- Extract Reports1 (Code)  
- Update Task Status1 (Google Sheets)  
- Format Confirmation1 (Code)  
- Confirm Report Save1 (Telegram)  

**Node Details:**

- **Check process_photo3**  
  - Filters for 'process_photo' action.

- **Extract photo data4**  
  - Extracts photo metadata and Task ID from reply message, validates presence and size limits.

- **Check error2**  
  - Checks if extraction resulted in error.

- **Photo error message4**  
  - Sends error message if photo processing fails.

- **Get and download file4**  
  - Downloads the photo file from Telegram.

- **Check file presence4**  
  - Validates if file binary data was received.

- **Upload to Google Drive4**  
  - Uploads photo to Google Drive folder.

- **Handle retrieval error4**  
  - Handles cases when file download fails, saves Telegram file_id only.

- **Merge file streams2**  
  - Merges multiple file inputs for grouping.

- **Group files by task**  
  - Groups multiple photos by Task ID.

- **Check existing task1**  
  - Looks up if Task ID already exists in Photo Reports sheet.

- **Does task exist?1**  
  - Branches depending on task existence.

- **Prepare data for adding2**  
  - Prepares data for new task entry.

- **Prepare data for updating2**  
  - Prepares data to update existing task entry, merges old and new file info.

- **Add to reports table1 / Update row in table2**  
  - Writes new or updated photo report entries to Google Sheets.

- **Merge lookup and grouped files**  
  - Merges lookup results with grouped files for further processing.

- **Prepare confirmation**  
  - Forms confirmation message data.

- **Confirm photo receipt4**  
  - Sends confirmation message to Telegram user.

- **Check Task Owner1, Get Tasks for Check1, Check ID Match1, Check Process Report1, Extract Reports1, Update Task Status1, Format Confirmation1, Confirm Report Save1**  
  - Handle validation, extraction, and update of task text reports with confirmation messages.

**Failures:**  
- Missing or invalid Task ID in replies.  
- Large file sizes exceeding 20MB.  
- Google Drive upload failures.  
- Google Sheets read/write errors.  
- Telegram API limits or message errors.

---

#### 1.5 Read and Received Confirmations

**Overview:**  
Handles /read and /received commands from users to confirm receipt or reading of reminders and tasks, updates corresponding Google Sheets entries, and sends acknowledgments.

**Nodes Involved:**  
- Check read (If)  
- Read sheet for read (Google Sheets)  
- Prepare read updates (Code)  
- Filter read summary (If)  
- Filter valid read updates (If)  
- Update read status in table (Google Sheets)  
- Send read confirmation (Telegram)  
- Form read response (Code)  
- Check Received1 (If)  
- Get Tasks for Reading1 (Google Sheets)  
- Prepare Read Updates1 (Code)  
- Filter Valid Read Updates1 (If)  
- Update Read in Table1 (Google Sheets)  
- Send Received Confirmation (Telegram)  

**Node Details:**

- Reads relevant rows for the user and date, filters for reminders/tasks marked as sent but unread/unreceived.

- Prepares update batches to mark 'Reminder Read' or 'Received' status.

- Updates Google Sheets accordingly.

- Sends confirmation messages acknowledging the reading or receipt.

**Failures:**  
- Google Sheets read/write errors.  
- Missing or malformed command inputs.  
- Telegram message delivery failures.

---

#### 1.6 Geolocation Processing

**Overview:**  
Processes geolocation messages sent by users in reply to photo report reminders, extracts GPS coordinates, updates the "Photo Reports" sheet, and confirms receipt.

**Nodes Involved:**  
- Check Process Location2 (If)  
- Extract Location Data1 (Code)  
- Check Location Error1 (If)  
- Location Error Message1 (Telegram)  
- Find Task for GPS (Google Sheets)  
- Task Found? (If)  
- Update GPS in Table (Google Sheets)  
- Confirm GPS Receipt (Telegram)  
- Error - Task Not Found (Telegram)  

**Node Details:**

- Extracts latitude and longitude from Telegram location message.

- Parses Task ID from the replied reminder message.

- Validates task existence in Google Sheets.

- Updates GPS coordinates, map link, and timestamp in Photo Reports sheet.

- Confirms success or sends error messages accordingly.

**Failures:**  
- Missing or invalid geolocation data.  
- Reply not to correct reminder message.  
- Task ID not found in sheet.  
- Google Sheets update errors.

---

#### 1.7 Error Handling and User Guidance

**Overview:**  
Sends informative error messages to users for common mistakes like sending photos or geolocation without replying to reminders or invalid message formats.

**Nodes Involved:**  
- Check no_reply error (If)  
- Error message (Telegram)  
- Check error2 (If)  
- Photo error message4 (Telegram)  
- Check Location Error1 (If)  
- Location Error Message1 (Telegram)  

**Node Details:**

- Filters error states flagged in processing nodes.

- Sends user-friendly messages with instructions to correct the input.

**Failures:**  
- Telegram API failures.

---

#### 1.8 Data Grouping and Updates

**Overview:**  
Groups multiple uploaded files by task for batch processing and updates or creates corresponding entries in Google Sheets with file metadata and confirmation messages.

**Nodes Involved:**  
- Merge file streams2 (Merge)  
- Group files by task (Code)  
- Merge lookup and grouped files (Merge)  
- Does task exist?1 (If)  
- Prepare data for adding2 (Code)  
- Prepare data for updating2 (Code)  
- Add to reports table1 (Google Sheets)  
- Update row in table2 (Google Sheets)  
- Merge for confirmation (Merge)  
- Prepare confirmation (Code)  
- Confirm photo receipt4 (Telegram)  

**Node Details:**

- Merges multiple parallel streams of photo files.

- Aggregates file details for each task.

- Checks if the task already exists in the sheet.

- Prepares and applies either insert or update operations.

- Sends detailed confirmation messages with counts and status.

**Failures:**  
- Data inconsistency if merges fail.  
- Google Sheets concurrency issues.

---

### 3. Summary Table

| Node Name                       | Node Type                | Functional Role                               | Input Node(s)                    | Output Node(s)                                    | Sticky Note                                                                                                                                                                                                                      |
|--------------------------------|--------------------------|-----------------------------------------------|---------------------------------|--------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Check every minute - Tasks      | Cron                     | Trigger to check Tasks every minute            |                                 | Get tasks                                        | ## ⏰ AUTOMATIC REMINDERS  Tasks: Checks the task table every minute and sends reminders to executors. Priorities: 🔴 High, 🟡 Medium, 🟢 Low, ⚠️ Reminder                                                                         |
| Get tasks                      | Google Sheets            | Reads "Tasks" sheet data                        | Check every minute - Tasks       | Find and form messages - Tasks                    | ## 📊 DATA STRUCTURE  Google Sheets Tables: 1. Users 2. Photo Reports 3. Tasks. Key Fields: Task ID / Report ID, Executor ID, Status                                                                                                |
| Find and form messages - Tasks | Code                     | Parses tasks, forms reminder messages          | Get tasks                      | Are there messages to send? - Tasks               | ## ⏰ AUTOMATIC REMINDERS  Tasks: Checks the task table every minute and sends reminders to executors. Priorities: 🔴 High, 🟡 Medium, 🟢 Low, ⚠️ Reminder                                                                         |
| Are there messages to send? - Tasks | If                   | Checks if task reminder messages exist         | Find and form messages - Tasks  | Send to Telegram - Tasks / No messages - Tasks    | ## ⏰ AUTOMATIC REMINDERS  Tasks: Checks the task table every minute and sends reminders to executors. Priorities: 🔴 High, 🟡 Medium, 🟢 Low, ⚠️ Reminder                                                                         |
| Send to Telegram - Tasks        | Telegram                 | Sends task reminder messages                    | Are there messages to send? - Tasks | Prepare updates - Tasks                           | ## ⏰ AUTOMATIC REMINDERS  Tasks: Checks the task table every minute and sends reminders to executors. Priorities: 🔴 High, 🟡 Medium, 🟢 Low, ⚠️ Reminder                                                                         |
| Prepare updates - Tasks         | Code                     | Prepares updates for task reminders             | Send to Telegram - Tasks         | Are there updates? - Tasks                         | ## ⏰ AUTOMATIC REMINDERS  Tasks: Checks the task table every minute and sends reminders to executors. Priorities: 🔴 High, 🟡 Medium, 🟢 Low, ⚠️ Reminder                                                                         |
| Are there updates? - Tasks      | If                       | Checks if updates exist for tasks               | Prepare updates - Tasks          | Update status in table - Tasks / No updates - Tasks | ## ⏰ AUTOMATIC REMINDERS  Tasks: Checks the task table every minute and sends reminders to executors. Priorities: 🔴 High, 🟡 Medium, 🟢 Low, ⚠️ Reminder                                                                         |
| Update status in table - Tasks  | Google Sheets            | Updates task reminder status in sheet           | Are there updates? - Tasks       |                                                  | ## ⏰ AUTOMATIC REMINDERS  Tasks: Checks the task table every minute and sends reminders to executors. Priorities: 🔴 High, 🟡 Medium, 🟢 Low, ⚠️ Reminder                                                                         |
| No messages - Tasks             | NoOp                     | Terminal node when no task messages             | Are there messages to send? - Tasks |                                                  |                                                                                                                                                                                                                                 |
| No updates - Tasks              | NoOp                     | Terminal node when no task updates              | Are there updates? - Tasks       |                                                  |                                                                                                                                                                                                                                 |
| Check every minute - Photo      | Cron                     | Trigger to check Photo Reports every minute     |                                 | Get photo reports data                           | ## ⏰ AUTOMATIC REMINDERS PHOTO REPORTS  Photo journal: Checks the table every minute and sends reminders to send photo reports. Logic includes sending status update and personalized messages.                                  |
| Get photo reports data          | Google Sheets            | Reads "Photo Reports" sheet data                 | Check every minute - Photo       | Find and form messages - Photo                     | ## ⏰ AUTOMATIC REMINDERS PHOTO REPORTS  Photo journal: Checks the table every minute and sends reminders to send photo reports. Logic includes sending status update and personalized messages.                                  |
| Find and form messages - Photo | Code                     | Parses photo report data, forms reminder messages | Get photo reports data          | Are there messages to send?                        | ## ⏰ AUTOMATIC REMINDERS PHOTO REPORTS  Photo journal: Checks the table every minute and sends reminders to send photo reports. Logic includes sending status update and personalized messages.                                  |
| Are there messages to send?     | If                       | Checks if photo report reminder messages exist  | Find and form messages - Photo   | Send to Telegram - Photo / No messages             | ## ⏰ AUTOMATIC REMINDERS PHOTO REPORTS  Photo journal: Checks the table every minute and sends reminders to send photo reports. Logic includes sending status update and personalized messages.                                  |
| Send to Telegram - Photo        | Telegram                 | Sends photo report reminder messages             | Are there messages to send?      | Prepare Updates                                   | ## ⏰ AUTOMATIC REMINDERS PHOTO REPORTS  Photo journal: Checks the table every minute and sends reminders to send photo reports. Logic includes sending status update and personalized messages.                                  |
| Prepare Updates                | Code                     | Prepares updates for photo report reminders      | Send to Telegram - Photo         | Are there updates?                                | ## ⏰ AUTOMATIC REMINDERS PHOTO REPORTS  Photo journal: Checks the table every minute and sends reminders to send photo reports. Logic includes sending status update and personalized messages.                                  |
| Are there updates?             | If                       | Checks if updates exist for photo reports        | Prepare Updates                 | Update status in table - Photo / No updates        | ## ⏰ AUTOMATIC REMINDERS PHOTO REPORTS  Photo journal: Checks the table every minute and sends reminders to send photo reports. Logic includes sending status update and personalized messages.                                  |
| Update status in table - Photo  | Google Sheets            | Updates photo report reminder status in sheet    | Are there updates?               |                                                  | ## ⏰ AUTOMATIC REMINDERS PHOTO REPORTS  Photo journal: Checks the table every minute and sends reminders to send photo reports. Logic includes sending status update and personalized messages.                                  |
| No messages                    | NoOp                     | Terminal node when no photo messages              | Are there messages to send?      |                                                  |                                                                                                                                                                                                                                 |
| No updates                    | NoOp                     | Terminal node when no photo updates               | Are there updates?               |                                                  |                                                                                                                                                                                                                                 |
| Telegram Webhook - Receiving messages | Telegram Trigger    | Entry point for all Telegram messages             |                                 | Process incoming message1                         | ## 🚀 ENTRY POINT  Telegram Webhook handles all commands, photos, text reports, and geolocation messages from users.                                                                                                             |
| Process incoming message1      | Code                     | Parses and routes incoming Telegram messages      | Telegram Webhook - Receiving messages | Multiple downstream nodes based on action       | ## 🚀 ENTRY POINT  Telegram Webhook handles all commands, photos, text reports, and geolocation messages from users.                                                                                                             |
| Check register_user1           | If                       | Checks if user registration needed                | Process incoming message1        | Read users sheet1                                | ## 👤 REGISTRATION BLOCK  Handles new executor registration: checking user, requesting data, saving to Google Sheets, confirmation.                                                                                            |
| Read users sheet1             | Google Sheets            | Reads Users table                                  | Check register_user1             | Check registration1                              | ## 👤 REGISTRATION BLOCK  Handles new executor registration: checking user, requesting data, saving to Google Sheets, confirmation.                                                                                            |
| Check registration1            | Code                     | Verifies if user exists in Users                   | Read users sheet1               | Check new user1, Check existing user1            | ## 👤 REGISTRATION BLOCK  Handles new executor registration: checking user, requesting data, saving to Google Sheets, confirmation.                                                                                            |
| Check new user1                | If                       | Branches new user path                             | Check registration1             | Request registration data1 / (no output)         | ## 👤 REGISTRATION BLOCK  Handles new executor registration: checking user, requesting data, saving to Google Sheets, confirmation.                                                                                            |
| Request registration data1     | Code                     | Sends message requesting user data                 | Check new user1                 | Send message                                     | ## 👤 REGISTRATION BLOCK  Handles new executor registration: checking user, requesting data, saving to Google Sheets, confirmation.                                                                                            |
| Check existing user1           | If                       | Branches existing user path                        | Check registration1             | Generate welcome1                                | ## 👤 REGISTRATION BLOCK  Handles new executor registration: checking user, requesting data, saving to Google Sheets, confirmation.                                                                                            |
| Generate welcome1              | Code                     | Generates welcome message for new/existing users  | Check existing user1            | Send message                                     | ## 👤 REGISTRATION BLOCK  Handles new executor registration: checking user, requesting data, saving to Google Sheets, confirmation.                                                                                            |
| Confirm registration           | Code                     | Prepares confirmation message after registration   | Record new executor             | Send message2                                    | ## 👤 REGISTRATION BLOCK  Handles new executor registration: checking user, requesting data, saving to Google Sheets, confirmation.                                                                                            |
| Prepare data for recording     | Code                     | Adds registration timestamps                        | Check complete_registration1    | Record new executor                              | ## 👤 REGISTRATION BLOCK  Handles new executor registration: checking user, requesting data, saving to Google Sheets, confirmation.                                                                                            |
| Record new executor            | Google Sheets            | Appends new user data to Users                      | Prepare data for recording      | Confirm registration                             | ## 👤 REGISTRATION BLOCK  Handles new executor registration: checking user, requesting data, saving to Google Sheets, confirmation.                                                                                            |
| Send message                  | Telegram                 | Sends messages                                      | Multiple                       |                                                  | ## 🔧 SERVICE COMMANDS  Available commands: /start, /help, /status, /report, /read, /received                                                                                                                                    |
| Send message2                 | Telegram                 | Sends confirmation messages                         | Confirm registration           |                                                  | ## 👤 REGISTRATION BLOCK  Handles new executor registration: checking user, requesting data, saving to Google Sheets, confirmation.                                                                                            |
| Check process_photo3           | If                       | Filters photo processing actions                     | Process incoming message1        | Extract photo data4 / others                      | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Extract photo data4            | Code                     | Extracts photo metadata and Task ID                 | Check process_photo3            | Check error2                                     | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Check error2                  | If                       | Checks if photo extraction failed                    | Extract photo data4             | Photo error message4 / Get and download file4    | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Photo error message4           | Telegram                 | Sends error messages for photo processing            | Check error2                   |                                                  | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Get and download file4         | Telegram                 | Downloads photo file                                 | Check error2                   | Check file presence4                              | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Check file presence4          | If                       | Validates file download success                      | Get and download file4          | Upload to Google Drive4 / Handle retrieval error4 | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Upload to Google Drive4        | Google Drive             | Uploads photo to Google Drive                         | Check file presence4           | Prepare data for writing5                         | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Handle retrieval error4        | Code                     | Handles download errors, saves Telegram file ID     | Check file presence4           | Merge file streams2                              | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Merge file streams2            | Merge                    | Merges multiple photo file streams                   | Prepare data for writing5 / Handle retrieval error4 | Group files by task                          | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Group files by task            | Code                     | Groups photos by Task ID                              | Merge file streams2            | Check existing task1 / Merge lookup and grouped files | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Check existing task1           | Google Sheets            | Checks if task exists in Photo Reports sheet         | Group files by task            | Does task exist?1                                | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Does task exist?1              | If                       | Branches for task existence                            | Check existing task1           | Prepare data for updating2 / Prepare data for adding2 | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Prepare data for adding2       | Code                     | Prepares new task entry data                           | Does task exist?1              | Add to reports table1 / Merge for confirmation   | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Prepare data for updating2     | Code                     | Prepares update data for existing task entry          | Does task exist?1              | Merge for confirmation / Update row in table2    | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Add to reports table1          | Google Sheets            | Adds new photo report entry                            | Prepare data for adding2       |                                                  | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Update row in table2           | Google Sheets            | Updates existing photo report entry                    | Prepare data for updating2     |                                                  | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Merge lookup and grouped files | Merge                    | Merges task lookup and grouped file data              | Check existing task1 / Group files by task | Does task exist?1                            | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Prepare confirmation          | Code                     | Forms confirmation messages data                       | Merge for confirmation        | Confirm photo receipt4                            | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Confirm photo receipt4         | Telegram                 | Sends photo receipt confirmation to user              | Prepare confirmation          |                                                  | ## 📸 PHOTO REPORT PROCESSING  Main functionality: extraction, Drive upload, grouping, writing, confirmation.                                                                                                                    |
| Check Task Owner1              | If                       | Checks task ownership before processing report         | Process incoming message1      | Get Tasks for Check1                             | ## 📋 TEXT REPORT PROCESSING  Verifies task ownership, parses status, updates task sheet, sends confirmation.                                                                                                                   |
| Get Tasks for Check1           | Google Sheets            | Reads Tasks sheet for ownership check                  | Check Task Owner1             | Check ID Match1                                  | ## 📋 TEXT REPORT PROCESSING  Verifies task ownership, parses status, updates task sheet, sends confirmation.                                                                                                                   |
| Check ID Match1                | Code                     | Validates task ownership and composes report object    | Get Tasks for Check1           | Check Process Report1 / Check Command1           | ## 📋 TEXT REPORT PROCESSING  Verifies task ownership, parses status, updates task sheet, sends confirmation.                                                                                                                   |
| Check Process Report1          | If                       | Filters 'process_report' action                         | Check ID Match1               | Extract Reports1                                 | ## 📋 TEXT REPORT PROCESSING  Verifies task ownership, parses status, updates task sheet, sends confirmation.                                                                                                                   |
| Extract Reports1               | Code                     | Extracts individual reports from action                | Check Process Report1         | Update Task Status1                              | ## 📋 TEXT REPORT PROCESSING  Verifies task ownership, parses status, updates task sheet, sends confirmation.                                                                                                                   |
| Update Task Status1            | Google Sheets            | Updates task status and comments in Tasks sheet        | Extract Reports1              | Format Confirmation1                             | ## 📋 TEXT REPORT PROCESSING  Verifies task ownership, parses status, updates task sheet, sends confirmation.                                                                                                                   |
| Format Confirmation1           | Code                     | Groups reports by executor and formats confirmation     | Update Task Status1           | Confirm Report Save1                             | ## 📋 TEXT REPORT PROCESSING  Verifies task ownership, parses status, updates task sheet, sends confirmation.                                                                                                                   |
| Confirm Report Save1           | Telegram                 | Sends confirmation message after saving task report    | Format Confirmation1          |                                                  | ## 📋 TEXT REPORT PROCESSING  Verifies task ownership, parses status, updates task sheet, sends confirmation.                                                                                                                   |
| Check read                    | If                       | Filters /read action                                   | Process incoming message1      | Read sheet for read                              | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Read sheet for read            | Google Sheets            | Reads Photo Reports for read confirmation               | Check read                   | Prepare read updates                             | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Prepare read updates           | Code                     | Prepares updates to mark reminders as read              | Read sheet for read           | Filter read summary                              | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Filter read summary            | If                       | Filters summary update data                              | Prepare read updates          | Form read response                               | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Form read response             | Code                     | Forms confirmation message for /read                     | Filter read summary           | Send read confirmation                           | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Update read status in table    | Google Sheets            | Updates read status columns in Photo Reports             | Filter valid read updates     | Send read confirmation                           | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Send read confirmation         | Telegram                 | Sends acknowledgment of /read command                    | Form read response            |                                                  | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Check Received1                | If                       | Filters /received command                                | Process incoming message1      | Get Tasks for Reading1                           | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Get Tasks for Reading1         | Google Sheets            | Reads Tasks for received confirmation                     | Check Received1              | Prepare Read Updates1                            | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Prepare Read Updates1          | Code                     | Prepares updates for received tasks                       | Get Tasks for Reading1        | Filter Valid Read Updates1                        | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Filter Valid Read Updates1     | If                       | Filters valid received updates                            | Prepare Read Updates1         | Update Read in Table1                            | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Update Read in Table1          | Google Sheets            | Updates read status for tasks                             | Filter Valid Read Updates1    | Send Received Confirmation                       | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Send Received Confirmation     | Telegram                 | Sends acknowledgment of /received command                 | Update Read in Table1         |                                                  | ## 🔄 PHOTO PROCESSING FLOW  1. User replies to reminder with photo 2. Extract Task ID 3. Upload 4. Check existing 5. Add or create entry 6. Send confirmation                                                                    |
| Check Process Location2        | If                       | Filters geolocation processing                            | Process incoming message1      | Extract Location Data1                           | ## 📍 GEOLOCATION PROCESSING  GPS Coordinates: Link to report, save coordinates and map link, update Google Sheets record.                                                                                                         |
| Extract Location Data1         | Code                     | Extracts geolocation data and Task ID                     | Check Process Location2       | Check Location Error1 / Find Task for GPS        | ## 📍 GEOLOCATION PROCESSING  GPS Coordinates: Link to report, save coordinates and map link, update Google Sheets record.                                                                                                         |
| Check Location Error1          | If                       | Checks for geolocation errors                             | Extract Location Data1        | Location Error Message1 / Find Task for GPS       | ## 📍 GEOLOCATION PROCESSING  GPS Coordinates: Link to report, save coordinates and map link, update Google Sheets record.                                                                                                         |
| Location Error Message1        | Telegram                 | Sends error message for geolocation processing            | Check Location Error1         |                                                  | ## 📍 GEOLOCATION PROCESSING  GPS Coordinates: Link to report, save coordinates and map link, update Google Sheets record.                                                                                                         |
| Find Task for GPS              | Google Sheets            | Checks if Photo Report task exists                         | Check Location Error1         | Task Found?                                       | ## 📍 GEOLOCATION PROCESSING  GPS Coordinates: Link to report, save coordinates and map link, update Google Sheets record.                                                                                                         |
| Task Found?                   | If                       | Branches based on task existence                           | Find Task for GPS             | Update GPS in Table / Error - Task Not Found      | ## 📍 GEOLOCATION PROCESSING  GPS Coordinates: Link to report, save coordinates and map link, update Google Sheets record.                                                                                                         |
| Update GPS in Table            | Google Sheets            | Updates GPS coordinates and map link                       | Task Found?                  | Confirm GPS Receipt                              | ## 📍 GEOLOCATION PROCESSING  GPS Coordinates: Link to report, save coordinates and map link, update Google Sheets record.                                                                                                         |
| Confirm GPS Receipt            | Telegram                 | Confirms successful GPS receipt                            | Update GPS in Table           |                                                  | ## 📍 GEOLOCATION PROCESSING  GPS Coordinates: Link to report, save coordinates and map link, update Google Sheets record.                                                                                                         |
| Error - Task Not Found         | Telegram                 | Sends error message if task not found for GPS              | Task Found?                  |                                                  | ## 📍 GEOLOCATION PROCESSING  GPS Coordinates: Link to report, save coordinates and map link, update Google Sheets record.                                                                                                         |
| Check no_reply error           | If                       | Filters errors due to missing reply to reminders           | Process incoming message1      | Error message                                    | ## ⚠️ IMPORTANT NOTES  1. Task ID = Report ID; 2. File Grouping; 3. Recurring Reminders; 4. Time Validation (Europe/Minsk timezone)                                                                                                |
| Error message                 | Telegram                 | Sends error messages for no-reply cases                     | Check no_reply error          |                                                  | ## ⚠️ IMPORTANT NOTES  1. Task ID = Report ID; 2. File Grouping; 3. Recurring Reminders; 4. Time Validation (Europe/Minsk timezone)                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron nodes to trigger checks every minute:**  
   - "Check every minute - Tasks" with schedule "everyMinute".  
   - "Check every minute - Photo" with schedule "everyMinute".

2. **Create Google Sheets nodes to read data:**  
   - "Get tasks": Read from "Tasks" sheet in the project spreadsheet.  
   - "Get photo reports data": Read from "Photo Reports" sheet.

3. **Create Code nodes to parse and form reminder messages:**  
   - "Find and form messages - Tasks": Parse task rows, filter by date/time, create reminders grouped by executors.  
   - "Find and form messages - Photo": Similar for photo reports, includes logic for repeated reminders.

4. **Create If nodes to check if messages exist:**  
   - "Are there messages to send? - Tasks" and "Are there messages to send?" for photos.

5. **Create Telegram send nodes:**  
   - "Send to Telegram - Tasks" and "Send to Telegram - Photo" configured with Telegram API credentials, sending HTML messages.

6. **Create Code nodes to prepare update data for Google Sheets:**  
   - "Prepare updates - Tasks" and "Prepare Updates" for photos. These extract row indexes and prepare status updates.

7. **Create If nodes to check if updates exist:**  
   - "Are there updates? - Tasks" and "Are there updates?" for photos.

8. **Create Google Sheets update nodes:**  
   - "Update status in table - Tasks" updates "Tasks" sheet with reminder sent status.  
   - "Update status in table - Photo" similarly for "Photo Reports".

9. **Create NoOp nodes as fallbacks:**  
   - "No messages - Tasks", "No updates - Tasks", "No messages", "No updates".

10. **Create Telegram Trigger node:**  
    - "Telegram Webhook - Receiving messages" listens for message, photo, and document updates.

11. **Create Code node "Process incoming message1":**  
    - Parses incoming messages for commands, photos, geolocation, text reports, and registration data.  
    - Routes to different actions like registering user, sending help, processing photos etc.

12. **Create Registration flow:**  
    - "Check register_user1" (If) filtering for register_user action.  
    - "Read users sheet1" to read "Users" sheet.  
    - "Check registration1" (Code) to check if user exists.  
    - "Check new user1" (If) branches new user path.  
    - "Request registration data1" (Code) to send prompt message.  
    - "Check existing user1" (If) branches existing user.  
    - "Generate welcome1" (Code) to generate welcome messages.  
    - "Prepare data for recording" adds registration date/time.  
    - "Record new executor" appends user to Google Sheets.  
    - Telegram send nodes to deliver messages.

13. **Create photo report processing flow:**  
    - "Check process_photo3" (If) filters photo processing.  
    - "Extract photo data4" (Code) extracts Task ID and metadata.  
    - "Check error2" (If) handles errors.  
    - "Photo error message4" sends error messages.  
    - "Get and download file4" downloads photo file from Telegram.  
    - "Check file presence4" verifies file presence.  
    - "Upload to Google Drive4" uploads photo to Drive.  
    - "Handle retrieval error4" handles download failures.  
    - "Merge file streams2" merges photo file streams.  
    - "Group files by task" groups photos by Task ID.  
    - "Check existing task1" looks up existing photo report by Task ID.  
    - "Does task exist?1" branches update vs new.  
    - "Prepare data for adding2" prepares new entry data.  
    - "Prepare data for updating2" prepares update data.  
    - "Add to reports table1" inserts new entries.  
    - "Update row in table2" updates existing entries.  
    - "Merge lookup and grouped files" merges data.  
    - "Prepare confirmation" forms confirmation message data.  
    - "Confirm photo receipt4" sends confirmation message.

14. **Create text report processing flow:**  
    - "Check Task Owner1" verifies task ownership.  
    - "Get Tasks for Check1" reads tasks sheet.  
    - "Check ID Match1" validates task ID ownership.  
    - "Check Process Report1" filters action.  
    - "Extract Reports1" extracts reports.  
    - "Update Task Status1" updates task sheet.  
    - "Format Confirmation1" formats confirmation.  
    - "Confirm Report Save1" sends confirmation.

15. **Create read and received confirmation flows:**  
    - For photo reports: "Check read", "Read sheet for read", "Prepare read updates", "Filter read summary", "Form read response", "Update read status in table", "Send read confirmation".  
    - For tasks: "Check Received1", "Get Tasks for Reading1", "Prepare Read Updates1", "Filter Valid Read Updates1", "Update Read in Table1", "Send Received Confirmation".

16. **Create geolocation processing flow:**  
    - "Check Process Location2" (If) filters geolocation action.  
    - "Extract Location Data1" extracts GPS data and task ID.  
    - "Check Location Error1" handles errors.  
    - "Location Error Message1" sends error messages.  
    - "Find Task for GPS" looks up task.  
    - "Task Found?" branches update or error.  
    - "Update GPS in Table" updates GPS data in sheet.  
    - "Confirm GPS Receipt" sends confirmation.

17. **Create error handling nodes:**  
    - "Check no_reply error" (If) filters no-reply errors.  
    - "Error message" sends user guidance.  
    - "Check error2" and "Photo error message4" for photo errors.  
    - "Check Location Error1" and "Location Error Message1" for GPS errors.

18. **Add Sticky Notes to document blocks and logic:**  
    - Add explanatory sticky notes as per the original workflow for clarity.

19. **Configure all credentials:**  
    - Google Sheets OAuth2 for accessing sheets.  
    - Google Drive OAuth2 for file uploads.  
    - Telegram API credentials with appropriate bot tokens and webhook setup.

20. **Set webhook URLs and enable active workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow by Artem Boiko at DataDrivenConstruction.io - https://datadrivenconstruction.io/                                                                | Project credits and source reference.                                                             |
| Priorities are indicated with emojis: 🔴 High, 🟡 Medium, 🟢 Low, ⚠️ Reminder                                                                           | Used throughout reminders for tasks.                                                             |
| Time validation accounts for Europe/Minsk timezone                                                                                                     | Important for correct timing of reminders.                                                        |
| Telegram bot commands supported: /start, /help, /status, /report, /read, /received                                                                      | Commands for user interaction.                                                                    |
| Photo files max size: 20 MB                                                                                                                             | Enforced during photo upload processing.                                                         |
| Users must reply to reminder messages when sending photos or geolocation                                                                                 | This ensures correct association with tasks or reports.                                          |
| Google Sheets tables structure: Users, Photo Reports, Tasks                                                                                            | Key tables used for data storage.                                                                |
| Use row_number for precise row updates in Google Sheets                                                                                                | Ensures accurate status updates without conflicts.                                               |
| Sticky notes in workflow provide detailed explanations for blocks and logic                                                                            | Helpful for maintainers and developers.                                                          |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---