Employee Attendance Tracker with Telegram Bot and Google Sheets

https://n8nworkflows.xyz/workflows/employee-attendance-tracker-with-telegram-bot-and-google-sheets-8335


# Employee Attendance Tracker with Telegram Bot and Google Sheets

### 1. Workflow Overview

This workflow, titled **Employee Attendance Tracker with Telegram Bot and Google Sheets**, enables employees to mark their attendance (check-in and check-out) and view daily attendance status via a Telegram bot. It integrates Telegram for user interaction and Google Sheets as a backend database to store and manage employee and attendance data.

The workflow is structured into the following logical blocks:

- **1.1 Telegram Interaction (Input Reception & Menu Navigation):** Handles Telegram updates (messages and callback queries), detects commands like `/start` or `/menu`, and presents users with interactive menus to select attendance actions or view status.

- **1.2 Attendance Action Processing:** Processes the user's choice to check-in or check-out. It verifies employee registration, checks for duplicate attendance entries for the day, and appends valid attendance records to Google Sheets.

- **1.3 Attendance Status Reporting:** Retrieves and formats the user’s attendance status for the current day and sends it back via Telegram.

- **1.4 Employee Verification:** Validates whether the Telegram user is a registered employee by querying the Google Sheets Employee list before allowing attendance actions or status queries.

- **1.5 Duplicate Attendance Prevention:** Checks if the user has already recorded a check-in or check-out for the day to avoid multiple logs.

Each block consists of dedicated nodes responsible for discrete tasks, connected in a clear, condition-driven sequence.

---

### 2. Block-by-Block Analysis

#### 1.1 Telegram Interaction (Input Reception & Menu Navigation)

**Overview:**  
This block listens for Telegram messages and callback queries. It detects commands `/start` or `/menu` and responds by sending a main menu with options for attendance check-in/out or viewing attendance status.

**Nodes Involved:**  
- Telegram Trigger  
- IF /start|/menu  
- Send Main Menu  
- IF menu_attendance  
- Send Check In/Out Menu  
- IF menu_attendance_status

**Node Details:**

- **Telegram Trigger**  
  - *Type:* Telegram Trigger  
  - *Role:* Entry point receiving Telegram messages and callback queries.  
  - *Configuration:* Listens for `"message"` and `"callback_query"` update types. Uses Telegram API credentials.  
  - *Inputs:* None (webhook)  
  - *Outputs:* Emits Telegram update JSON for use downstream  
  - *Potential Failures:* Network issues, invalid webhook setup, Telegram API errors.

- **IF /start|/menu**  
  - *Type:* IF Node  
  - *Role:* Checks if the incoming message text matches `/start` or `/menu` command using regex.  
  - *Configuration:* Condition tests message or channel_post text with regex `^/(start|menu)$`.  
  - *Inputs:* Telegram Trigger output  
  - *Outputs:* True branch to show main menu; False ignored.  
  - *Edge Cases:* Messages without text, unexpected commands.

- **Send Main Menu**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Sends a Telegram message presenting the main menu with inline buttons: "Check In/Out" and "Today’s Attendance Status."  
  - *Configuration:* Sends text "Please select an option:" with inline keyboard buttons linked to callback data (`menu_attendance` and `menu_attendance_status`). Uses Telegram credentials.  
  - *Inputs:* IF /start|/menu (true)  
  - *Outputs:* None  
  - *Edge Cases:* Invalid chat IDs, Telegram API limits.

- **IF menu_attendance**  
  - *Type:* IF Node  
  - *Role:* Detects if the callback query data is `menu_attendance` (user selected Check In/Out menu).  
  - *Configuration:* Compares callback query data string equals `menu_attendance`.  
  - *Inputs:* Telegram Trigger output  
  - *Outputs:* True branch to send Check In/Out menu.

- **Send Check In/Out Menu**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Sends Check In/Out menu with inline buttons "Check In" and "Check Out."  
  - *Configuration:* Text "Pilih absen:" (Choose attendance), buttons with callback data `checkin` and `checkout`.  
  - *Inputs:* IF menu_attendance (true)  
  - *Outputs:* None

- **IF menu_attendance_status**  
  - *Type:* IF Node  
  - *Role:* Detects if the callback query data is `menu_attendance_status` (user selected to view attendance status).  
  - *Configuration:* Compares callback query data string equals `menu_attendance_status`.  
  - *Inputs:* Telegram Trigger output  
  - *Outputs:* True branch triggers attendance status retrieval block.

---

#### 1.2 Attendance Action Processing

**Overview:**  
Processes check-in or check-out commands from the user. Validates employee registration, checks for duplicates, appends attendance record, and sends confirmation or error messages.

**Nodes Involved:**  
- IF callback_query  
- Sheets Read (employee)1  
- IF Employee  
- Sheets Read (duplicate attendance)  
- IF duplicate attendance  
- Reply Duplicate Attendance  
- Sheets Append (Attendance)  
- Reply Attendance OK  
- Reply Not Employee1

**Node Details:**

- **IF callback_query**  
  - *Type:* IF Node  
  - *Role:* Checks if callback data equals `checkin` or `checkout`.  
  - *Configuration:* Logical OR for callback data `"checkin"` or `"checkout"`.  
  - *Inputs:* Telegram Trigger output  
  - *Outputs:* True branch proceeds to employee verification.

- **Sheets Read (employee)1**  
  - *Type:* Google Sheets Node (Read)  
  - *Role:* Reads Employee sheet filtering by Telegram username from callback query to validate the user as employee.  
  - *Configuration:* Filters where `username_telegram` equals Telegram user’s username. Sheet "Employee" in specified Google Sheet.  
  - *Inputs:* IF callback_query (true)  
  - *Outputs:* Employee data if found.

- **IF Employee**  
  - *Type:* IF Node  
  - *Role:* Checks if employee data exists (non-empty).  
  - *Inputs:* Sheets Read (employee)1 output  
  - *Outputs:* True branch proceeds to duplicate attendance check; false branch replies with not-employee message.

- **Sheets Read (duplicate attendance)**  
  - *Type:* Google Sheets Node (Read)  
  - *Role:* Checks if attendance record already exists for the user, for the current day, and for the same attendance type (checkin or checkout).  
  - *Configuration:* Filters on `username_telegram`, today’s date, and `attendance_type` (from callback data).  
  - *Inputs:* IF Employee (true)  
  - *Outputs:* Attendance records if any.

- **IF duplicate attendance**  
  - *Type:* IF Node  
  - *Role:* Determines if duplicate attendance record exists (non-empty).  
  - *Inputs:* Sheets Read (duplicate attendance) output  
  - *Outputs:* True branch sends duplicate warning; false branch appends new record.

- **Reply Duplicate Attendance**  
  - *Type:* Telegram Node  
  - *Role:* Sends message to user that their attendance (check-in/check-out) is already recorded.  
  - *Configuration:* Message includes dynamic attendance type from callback data.  
  - *Inputs:* IF duplicate attendance (true)  
  - *Outputs:* None

- **Sheets Append (Attendance)**  
  - *Type:* Google Sheets Node (Append)  
  - *Role:* Appends a new attendance record with date, time, timestamp, attendance type, employee id, Telegram username, and full name.  
  - *Configuration:* Uses current date/time; maps employee data fields from previous read.  
  - *Inputs:* IF duplicate attendance (false)  
  - *Outputs:* Confirmation of append

- **Reply Attendance OK**  
  - *Type:* Telegram Node  
  - *Role:* Sends confirmation message for successful check-in or check-out.  
  - *Configuration:* Message varies by attendance type: "Check-in recorded. ⏰" or "Check-out recorded. 🏁".  
  - *Inputs:* Sheets Append (Attendance) output  
  - *Outputs:* None

- **Reply Not Employee1**  
  - *Type:* Telegram Node  
  - *Role:* Sends error message if user is not registered as employee.  
  - *Inputs:* IF Employee (false)  
  - *Outputs:* None

---

#### 1.3 Attendance Status Reporting

**Overview:**  
Fetches the user’s attendance records for the current day, aggregates check-in and check-out status, formats a summary message, and sends it back via Telegram.

**Nodes Involved:**  
- IF menu_attendance_status  
- Sheets Read (employee)  
- IF employee  
- Sheets Read (Attendance Status)  
- Aggregate  
- Format Attendance Status  
- Reply Attendance Status  
- Reply Not Employee

**Node Details:**

- **IF menu_attendance_status**  
  - *Type:* IF Node  
  - *Role:* Detects if callback data equals `menu_attendance_status`.  
  - *Inputs:* Telegram Trigger output  
  - *Outputs:* True branch triggers employee verification.

- **Sheets Read (employee)**  
  - *Type:* Google Sheets Node (Read)  
  - *Role:* Reads Employee sheet filtering by Telegram username to validate user.  
  - *Inputs:* IF menu_attendance_status (true)  
  - *Outputs:* Employee data.

- **IF employee**  
  - *Type:* IF Node  
  - *Role:* Checks if employee record exists (non-empty).  
  - *Inputs:* Sheets Read (employee) output  
  - *Outputs:* True branch proceeds to attendance status read; false branch sends error message.

- **Sheets Read (Attendance Status)**  
  - *Type:* Google Sheets Node (Read)  
  - *Role:* Retrieves attendance records for the user for the current day. Filters by username and date (formatted dd-MM-yyyy).  
  - *Inputs:* IF employee (true)  
  - *Outputs:* Attendance data for aggregation.

- **Aggregate**  
  - *Type:* Aggregate Node  
  - *Role:* Aggregates the attendance data array for processing.  
  - *Inputs:* Sheets Read (Attendance Status) output  
  - *Outputs:* Aggregated data for formatting.

- **Format Attendance Status**  
  - *Type:* Function Item Node  
  - *Role:* Analyzes attendance data for check-in and check-out presence, constructs a summary message with checkmarks or crosses.  
  - *Configuration:* Checks existence of `checkin` and `checkout` attendance types; generates message like:  
    ```
    Today’s Attendance Status:
    Check In: ✅ or ❌
    Check Out: ✅ or ❌
    ```  
  - *Inputs:* Aggregate output  
  - *Outputs:* JSON object with `text` property for Telegram message.

- **Reply Attendance Status**  
  - *Type:* Telegram Node  
  - *Role:* Sends the formatted attendance status message to the user.  
  - *Inputs:* Format Attendance Status output  
  - *Outputs:* None

- **Reply Not Employee**  
  - *Type:* Telegram Node  
  - *Role:* Sends error message if user not registered as employee.  
  - *Inputs:* IF employee (false)  
  - *Outputs:* None

---

#### 1.4 Employee Verification

**Overview:**  
Performs employee validation by querying the Employee sheet in Google Sheets and verifying the Telegram username exists.

**Nodes Involved:**  
- Sheets Read (employee)  
- IF employee  
- Sheets Read (employee)1  
- IF Employee

**Node Details:**  

- Both **Sheets Read (employee)** and **Sheets Read (employee)1** perform the identical function of reading the Employee sheet filtered by Telegram username, used in different branches (attendance status, attendance action).  
- **IF employee** and **IF Employee** nodes check non-empty data to confirm employee existence.  
- Failure to validate results in sending a "You are not listed as an employee. Contact the administrator." message.

---

#### 1.5 Duplicate Attendance Prevention

**Overview:**  
Ensures that employees cannot record multiple check-ins or check-outs on the same day by checking existing attendance records.

**Nodes Involved:**  
- Sheets Read (duplicate attendance)  
- IF duplicate attendance  
- Reply Duplicate Attendance

**Node Details:**  

- **Sheets Read (duplicate attendance)** queries the Attendance sheet filtering by username, current date, and attendance type to find existing records.  
- **IF duplicate attendance** checks if the query returned any data (non-empty).  
- If duplicate found, **Reply Duplicate Attendance** sends warning: "Your check-in/check-out has already been recorded ⚠️".  
- If no duplicate, workflow continues to append new attendance record.

---

### 3. Summary Table

| Node Name                  | Node Type                  | Functional Role                        | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                      |
|----------------------------|----------------------------|-------------------------------------|---------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------|
| Telegram Trigger           | Telegram Trigger           | Entry point for Telegram updates    | None                            | IF /start|/menu, IF menu_attendance, IF menu_attendance_status, IF callback_query |                                                                                                |
| IF /start|/menu            | IF                         | Detect `/start` or `/menu` command  | Telegram Trigger                | Send Main Menu                    |                                                                                                |
| Send Main Menu             | Telegram                   | Sends main menu with options        | IF /start|/menu                  | None                             | ## Send Menu Button                                                                             |
| IF menu_attendance         | IF                         | Detect "Check In/Out" menu selection| Telegram Trigger                | Send Check In/Out Menu           |                                                                                                |
| Send Check In/Out Menu     | Telegram                   | Sends Check In/Out buttons          | IF menu_attendance              | None                             | ## Send Check In/Out Button                                                                    |
| IF menu_attendance_status  | IF                         | Detect "Attendance Status" selection| Telegram Trigger                | Sheets Read (employee)           |                                                                                                |
| Sheets Read (employee)     | Google Sheets Read         | Verify employee for attendance status| IF menu_attendance_status       | IF employee                     | ## Verify Employee                                                                             |
| IF employee                | IF                         | Check if employee exists             | Sheets Read (employee)          | Sheets Read (Attendance Status), Reply Not Employee |                                                                                                |
| Sheets Read (Attendance Status) | Google Sheets Read   | Read today's attendance for user    | IF employee                    | Aggregate                      |                                                                                                |
| Aggregate                 | Aggregate                  | Aggregate attendance data            | Sheets Read (Attendance Status)| Format Attendance Status         |                                                                                                |
| Format Attendance Status  | Function Item              | Format attendance summary message   | Aggregate                      | Reply Attendance Status          | ## Format & Send Today's Attendance Status                                                     |
| Reply Attendance Status   | Telegram                   | Send attendance status message      | Format Attendance Status       | None                           |                                                                                                |
| Reply Not Employee        | Telegram                   | Notify user not an employee          | IF employee (false)            | None                           |                                                                                                |
| IF callback_query         | IF                         | Detect check-in or check-out action | Telegram Trigger                | Sheets Read (employee)1          |                                                                                                |
| Sheets Read (employee)1   | Google Sheets Read         | Verify employee for attendance action| IF callback_query              | IF Employee                    | ## Verify Employee                                                                             |
| IF Employee               | IF                         | Check employee existence             | Sheets Read (employee)1        | Sheets Read (duplicate attendance), Reply Not Employee1 |                                                                                                |
| Sheets Read (duplicate attendance) | Google Sheets Read  | Check for duplicate attendance      | IF Employee                   | IF duplicate attendance          |                                                                                                |
| IF duplicate attendance   | IF                         | Determine if attendance is duplicate| Sheets Read (duplicate attendance)| Reply Duplicate Attendance, Sheets Append (Attendance) | ## Check for Duplicate Attendance                                                             |
| Reply Duplicate Attendance| Telegram                   | Warn user about duplicate attendance| IF duplicate attendance (true) | None                           |                                                                                                |
| Sheets Append (Attendance)| Google Sheets Append       | Append new attendance record         | IF duplicate attendance (false)| Reply Attendance OK             |                                                                                                |
| Reply Attendance OK       | Telegram                   | Confirm successful attendance record | Sheets Append (Attendance)     | None                           |                                                                                                |
| Reply Not Employee1       | Telegram                   | Notify user not an employee          | IF Employee (false)            | None                           |                                                                                                |
| Aggregate                 | Aggregate                  | Aggregate node                       | Sheets Read (Attendance Status)| Format Attendance Status         |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger node**  
   - Type: Telegram Trigger  
   - Configure to listen for `message` and `callback_query` updates.  
   - Set credentials with your Telegram Bot API token.

2. **Add IF node "IF /start|/menu"**  
   - Condition: Check if message text or channel_post text matches regex `^/(start|menu)$`.  
   - Input: Telegram Trigger.

3. **Add Telegram node "Send Main Menu"**  
   - Text: "Please select an option:"  
   - Chat ID: `{{$json.message.chat.id}}`  
   - Reply markup: Inline keyboard with buttons:  
     - Text: "🕘 Check In/Out", callback_data: `menu_attendance`  
     - Text: "📊 Today’s Attendance Status", callback_data: `menu_attendance_status`  
   - Input: IF /start|/menu (true branch).

4. **Add IF node "IF menu_attendance"**  
   - Condition: Check if callback_query.data equals `menu_attendance`  
   - Input: Telegram Trigger.

5. **Add Telegram node "Send Check In/Out Menu"**  
   - Text: "Pilih absen:"  
   - Chat ID: `{{$json.callback_query.message.chat.id}}`  
   - Inline keyboard buttons:  
     - "✅ Check In", callback_data: `checkin`  
     - "🚪 Check Out", callback_data: `checkout`  
   - Input: IF menu_attendance (true branch).

6. **Add IF node "IF menu_attendance_status"**  
   - Condition: callback_query.data equals `menu_attendance_status`  
   - Input: Telegram Trigger.

7. **Add Google Sheets Read node "Sheets Read (employee)"** (used for attendance status)  
   - Document ID: Your Google Sheet ID  
   - Sheet Name: Employee sheet  
   - Filter: `username_telegram` equals `{{$json.callback_query.from.username}}`  
   - Input: IF menu_attendance_status (true branch).  
   - Set Google Sheets OAuth2 credentials.

8. **Add IF node "IF employee"**  
   - Condition: Check if data array is non-empty (employee exists).  
   - Input: Sheets Read (employee).

9. **Add Google Sheets Read node "Sheets Read (Attendance Status)"**  
   - Document ID: same sheet  
   - Sheet: Attendance sheet  
   - Filters:  
     - `username_telegram` equals `{{$json.username_telegram}}` (from employee data)  
     - `date` equals `{{$today.format('dd-MM-yyyy')}}`  
   - Input: IF employee (true branch).

10. **Add Aggregate node**  
    - Operation: aggregate all item data  
    - Input: Sheets Read (Attendance Status).

11. **Add Function Item node "Format Attendance Status"**  
    - Code:  
      ```js
      const rows = Array.isArray($json.data) ? $json.data : [];
      let checkin = rows.some(r => (r.attendance_type||'')==='checkin');
      let checkout = rows.some(r => (r.attendance_type||'')==='checkout');
      let msg = 'Today’s Attendance Status:\n';
      msg += `Check In: ${checkin ? '✅' : '❌'}\n`;
      msg += `Check Out: ${checkout ? '✅' : '❌'}`;
      return { text: msg };
      ```  
    - Input: Aggregate.

12. **Add Telegram node "Reply Attendance Status"**  
    - Text: `{{$json.text}}`  
    - Chat ID: `{{$json.callback_query.message.chat.id}}` (from IF menu_attendance_status node)  
    - Input: Format Attendance Status.

13. **Add Telegram node "Reply Not Employee"**  
    - Text: "You are not listed as an employee. Contact the administrator."  
    - Chat ID: `{{$json.callback_query.message.chat.id}}` (from IF menu_attendance_status node)  
    - Input: IF employee (false branch).

14. **Add IF node "IF callback_query"**  
    - Condition: callback_query.data equals `checkin` or `checkout`  
    - Input: Telegram Trigger.

15. **Add Google Sheets Read node "Sheets Read (employee)1"** (used for attendance action)  
    - Same config as step 7  
    - Input: IF callback_query (true branch).

16. **Add IF node "IF Employee"**  
    - Same as step 8, checks if employee exists  
    - Input: Sheets Read (employee)1.

17. **Add Google Sheets Read node "Sheets Read (duplicate attendance)"**  
    - Document ID: Attendance sheet  
    - Filters:  
      - `username_telegram` equals `{{$json.username_telegram}}`  
      - `date` equals `{{$today.format('dd-MM-yyyy')}}`  
      - `attendance_type` equals `{{$json.callback_query.data}}` (checkin or checkout)  
    - Input: IF Employee (true branch).

18. **Add IF node "IF duplicate attendance"**  
    - Condition: Check if duplicate attendance data is non-empty  
    - Input: Sheets Read (duplicate attendance).

19. **Add Telegram node "Reply Duplicate Attendance"**  
    - Text: "Your {{$json.callback_query.data}} has already been recorded ⚠️"  
    - Chat ID: `{{$json.callback_query.message.chat.id}}`  
    - Input: IF duplicate attendance (true branch).

20. **Add Google Sheets Append node "Sheets Append (Attendance)"**  
    - Document ID: Attendance sheet  
    - Append new row with columns:  
      - `date`: `{{$today.format('dd-MM-yyyy')}}`  
      - `time`: `{{$now.format('HH:mm:ss')}}`  
      - `timestamp_iso`: `{{$now}}`  
      - `attendance_type`: `{{$json.callback_query.data}}`  
      - `id_employee`, `username_telegram`, `full_name`: from employee record (Sheets Read employee1)  
    - Input: IF duplicate attendance (false branch).

21. **Add Telegram node "Reply Attendance OK"**  
    - Text: Conditional message:  
      ```
      {{$json.callback_query.data === 'checkin' ? 'Check-in recorded. ⏰' : 'Check-out recorded. 🏁'}}
      ```  
    - Chat ID: `{{$json.callback_query.message.chat.id}}`  
    - Input: Sheets Append (Attendance).

22. **Add Telegram node "Reply Not Employee1"**  
    - Text: "You are not listed as an employee. Contact the administrator."  
    - Chat ID: `{{$json.callback_query.message.chat.id}}`  
    - Input: IF Employee (false branch).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                                      |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------|
| **Attendance Telegram App with Google Sheets**: Manage employee attendance via Telegram bot with Google Sheets backend. Includes Google Sheets template for Employee and Attendance sheets.                                                                                                                                                                                | Sticky Note content providing overview and setup instructions.                                                       |
| Features include check-in/out via Telegram, daily attendance status, employee validation, duplicate prevention, and Google Sheets integration.                                                                                                                                                                                                                            | Sticky Note content.                                                                                                 |
| Requirements: n8n platform, Telegram bot via BotFather, Google Sheets account.                                                                                                                                                                                                                                                                                             | Sticky Note content.                                                                                                 |
| Setup instructions include importing workflow, configuring Telegram and Google Sheets credentials, and using the provided Google Sheets template.                                                                                                                                                                                                                         | Sticky Note content.                                                                                                 |
| Google Sheets Template: [Copy the Google Sheets Template](https://docs.google.com/spreadsheets/d/1miqc4zpTecMwk_qNHgM17na2rDsWNpICIblKy44hwnw/edit?usp=sharing)                                                                                                                                                                                                        | External link in sticky note.                                                                                        |
| Example flow: User types `/menu` → bot displays menu → user selects check-in → workflow validates employee and duplicates → records attendance → bot replies confirmation.                                                                                                                                                                                                | Sticky Note content.                                                                                                 |
| Node groups with sticky notes clarify logical sections: sending main menu, sending check-in/out buttons, checking attendance status, verifying employee status, and handling duplicates.                                                                                                                                                                                    | Sticky Notes labeled Send Menu Button, Send Check In/Out Button, Check Today’s Attendance Status, Verify Employee, Check for Duplicate Attendance, Format & Send Today's Attendance Status. |

---

**Disclaimer:**  
The provided text is exclusively generated from an automated n8n workflow. It strictly complies with content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.