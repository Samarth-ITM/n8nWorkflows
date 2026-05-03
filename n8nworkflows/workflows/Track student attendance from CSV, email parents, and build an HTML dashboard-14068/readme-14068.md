Track student attendance from CSV, email parents, and build an HTML dashboard

https://n8nworkflows.xyz/workflows/track-student-attendance-from-csv--email-parents--and-build-an-html-dashboard-14068


# Track student attendance from CSV, email parents, and build an HTML dashboard

# 1. Workflow Overview

This workflow automates daily student attendance monitoring from CSV files, notifies parents by email when a student is absent, and generates a local HTML dashboard plus an updated historical CSV report.

Its primary use cases are:

- Daily school attendance tracking
- Parent alerting for absences
- Ongoing risk detection based on recent attendance patterns
- Building a visual dashboard from cumulative absence history

The workflow is designed to run automatically every weekday at 17:30, process only today’s attendance entries, enrich absent students with contact data, compute attendance risk metrics, send HTML email alerts, and then update reporting artifacts on disk.

## 1.1 Scheduled Trigger and Attendance Ingestion

This block starts the workflow on a cron schedule, reads the daily attendance CSV from the n8n file system, parses it, and keeps only the rows matching today’s date.

## 1.2 Absence Routing and Contact Enrichment

This block splits today’s records into absent vs. non-absent branches. If there are absent students, it loads the contact CSV and merges parent/contact information onto the attendance records.

## 1.3 Attendance Risk and Alert Preparation

This block calculates per-student metrics such as absences in the last 30 days, consecutive absence streak, attendance percentage, trend, and alert level. It also generates a styled HTML email body for parent notifications.

## 1.4 Parent Notification Delivery

This block sends absence emails to parents via SMTP using the generated subject and HTML content.

## 1.5 No-Absence Fallback Branch

If no student is marked absent for today, this block creates a minimal payload indicating that no absences occurred, allowing the reporting section to continue without attempting email delivery.

## 1.6 Dashboard and Historical Report Update

This block reads the existing historical report CSV, combines it with today’s new absence data, generates a full interactive HTML dashboard, converts report outputs into files, and writes both the updated CSV history and dashboard HTML back to disk.

---

# 2. Block-by-Block Analysis

## 2.1 Scheduled Trigger and Attendance Ingestion

### Overview
This block initiates the workflow every weekday at 17:30 and ingests attendance data from a CSV file stored in the n8n files directory. It parses the CSV and filters the dataset to today’s records only.

### Nodes Involved
- Recurring Time Trigger
- Student Attendance Data
- Extract Attendance CSV
- Date Filter

### Node Details

#### Recurring Time Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`; workflow entry point using a cron expression.
- **Configuration choices:**
  - Cron expression: `30 17 * * 1-5`
  - Runs Monday through Friday at 17:30.
- **Key expressions or variables used:** none.
- **Input and output connections:**
  - No input
  - Outputs to **Student Attendance Data**
- **Version-specific requirements:**
  - Type version `1.1`
- **Edge cases or potential failure types:**
  - Workflow must be active to run automatically.
  - Server timezone may differ from the intended school timezone; later nodes partly compensate using Asia/Kolkata, but the trigger itself runs according to the n8n/server schedule context.
- **Sub-workflow reference:** none

#### Student Attendance Data
- **Type and technical role:** `n8n-nodes-base.readWriteFile`; reads the source attendance CSV from local disk.
- **Configuration choices:**
  - Reads from `/home/node/.n8n-files/File_name.csv`
  - No extra options enabled.
- **Key expressions or variables used:** none.
- **Input and output connections:**
  - Input from **Recurring Time Trigger**
  - Output to **Extract Attendance CSV**
- **Version-specific requirements:**
  - Type version `1`
- **Edge cases or potential failure types:**
  - File not found
  - Permissions issue on the n8n files directory
  - Wrong file path left unchanged from template placeholder
  - Empty or malformed file
- **Sub-workflow reference:** none

#### Extract Attendance CSV
- **Type and technical role:** `n8n-nodes-base.spreadsheetFile`; parses binary CSV into JSON rows.
- **Configuration choices:**
  - Header row enabled
  - Raw data enabled
- **Key expressions or variables used:** none.
- **Input and output connections:**
  - Input from **Student Attendance Data**
  - Output to **Date Filter**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Invalid CSV structure
  - Missing headers such as `Date`, `StudentID`, `Status`
  - Raw cell types may vary depending on CSV contents
- **Sub-workflow reference:** none

#### Date Filter
- **Type and technical role:** `n8n-nodes-base.filter`; keeps only rows whose `Date` equals today.
- **Configuration choices:**
  - Compares `{{$json.Date}}` to `{{$now.toFormat('dd-MM-yyyy')}}`
  - Strict string equality
- **Key expressions or variables used:**
  - `={{ $json.Date }}`
  - `={{ $now.toFormat('dd-MM-yyyy') }}`
- **Input and output connections:**
  - Input from **Extract Attendance CSV**
  - Output to **IF: Is Absent?**
- **Version-specific requirements:**
  - Type version `2.3`
- **Edge cases or potential failure types:**
  - Date format mismatch between CSV and `dd-MM-yyyy`
  - Server timezone mismatch could cause “today” to be off by one day
  - Attendance CSV may contain Excel serial dates or ISO dates; this node does not normalize them before filtering
- **Sub-workflow reference:** none

---

## 2.2 Absence Routing and Contact Enrichment

### Overview
This block determines whether today’s attendance rows are marked absent and, when absences exist, enriches those rows with parent contact data from a second CSV.

### Nodes Involved
- IF: Is Absent?
- Students Contacts
- Extract Contacts CSV
- Merge Absent + Contacts

### Node Details

#### IF: Is Absent?
- **Type and technical role:** `n8n-nodes-base.if`; routes rows based on the `Status` field.
- **Configuration choices:**
  - Condition: `Status == "Absent"`
  - Strict, case-sensitive string comparison
- **Key expressions or variables used:**
  - `={{ $json.Status }}`
- **Input and output connections:**
  - Input from **Date Filter**
  - True output to **Students Contacts** and **Merge Absent + Contacts**
  - False output to **No Absences Today**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Rows with `absent`, `ABSENT`, or trailing spaces will fail the exact match
  - If multiple items flow through, true items and false items are split individually
  - The false branch is triggered for any non-absent row, not strictly “all present”; this matters if today’s filtered dataset contains both Present and Absent rows
- **Sub-workflow reference:** none

#### Students Contacts
- **Type and technical role:** `n8n-nodes-base.readWriteFile`; reads the student contacts CSV from local disk.
- **Configuration choices:**
  - Reads from `/home/node/.n8n-files/File_name.csv`
  - `executeOnce: true`, so contacts are loaded once per execution
- **Key expressions or variables used:** none.
- **Input and output connections:**
  - Input from **IF: Is Absent?** true branch
  - Output to **Extract Contacts CSV**
- **Version-specific requirements:**
  - Type version `1`
- **Edge cases or potential failure types:**
  - Same template placeholder path issue as the attendance file
  - Missing file or invalid permissions
  - File may not contain expected keys such as `StudentID`, `ParentName`, `Email`, `Phone`
- **Sub-workflow reference:** none

#### Extract Contacts CSV
- **Type and technical role:** `n8n-nodes-base.spreadsheetFile`; parses the contacts CSV into JSON.
- **Configuration choices:**
  - Header row enabled
- **Key expressions or variables used:** none.
- **Input and output connections:**
  - Input from **Students Contacts**
  - Output to **Merge Absent + Contacts** input 1
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Malformed CSV
  - Missing `StudentID` header
  - Duplicate contact rows for the same student may produce multiple merged records downstream
- **Sub-workflow reference:** none

#### Merge Absent + Contacts
- **Type and technical role:** `n8n-nodes-base.merge`; combines absent attendance records with contact records by student ID.
- **Configuration choices:**
  - Mode: `combine`
  - Merge by fields:
    - Input 0 field: `StudentID`
    - Input 1 field: `StudentID`
- **Key expressions or variables used:** none.
- **Input and output connections:**
  - Input 0 from **IF: Is Absent?** true branch
  - Input 1 from **Extract Contacts CSV**
  - Output to **Alert Logic Block**
- **Version-specific requirements:**
  - Type version `2.1`
- **Edge cases or potential failure types:**
  - No contact match for a student may drop or alter output depending on merge behavior
  - Duplicate `StudentID` in either file may create repeated combinations
  - Type mismatch between IDs such as numeric vs string may prevent matching
- **Sub-workflow reference:** none

---

## 2.3 Attendance Risk and Alert Preparation

### Overview
This block computes recent attendance metrics for each absent student, normalizes dates, assigns alert severity, and creates a styled HTML email payload to be sent to parents.

### Nodes Involved
- Alert Logic Block
- Generate Absence Email

### Node Details

#### Alert Logic Block
- **Type and technical role:** `n8n-nodes-base.code`; aggregates merged records by student and computes alert metrics.
- **Configuration choices:**
  - Uses JavaScript to:
    - Normalize dates from several formats
    - Set “today” using `Asia/Kolkata`
    - Group rows by `StudentID`
    - Compute:
      - `absencesLast30Days`
      - `attendancePct`
      - `consecutiveStreak`
      - `trend`
      - `alertLevel`
      - `alertReason`
    - Emit one summary item per student
- **Key expressions or variables used:**
  - `$input.all()`
  - Local helper functions:
    - `parseDate`
    - `normalizeDate`
  - Output fields include:
    - `studentId`
    - `name`
    - `parentName`
    - `email`
    - `phone`
    - `class`
    - `subject`
    - `teacher`
    - `date`
    - `absencesLast30Days`
    - `consecutiveStreak`
    - `attendancePct`
    - `trend`
    - `alertLevel`
    - `alertReason`
- **Input and output connections:**
  - Input from **Merge Absent + Contacts**
  - Outputs to **Generate Absence Email** and **Build Dashboard and Report**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - This node only sees the merged absent rows, not full historical daily attendance from the original CSV unless such rows are already present in the merged input. As written, the 30-day calculations depend entirely on what records reach this node.
  - Missing `StudentID` rows are skipped.
  - Invalid date strings may normalize poorly and distort rolling-window metrics.
  - If no email exists, downstream email sending may fail or be skipped manually.
  - Contact fallback values are generic (`Student`, `Parent`, empty strings).
- **Sub-workflow reference:** none

#### Generate Absence Email
- **Type and technical role:** `n8n-nodes-base.code`; builds the HTML email subject and body.
- **Configuration choices:**
  - Maps `alertLevel` to color palette, icon, and label
  - Uses trend labels/icons
  - Builds subject like:
    - `🔴 Absence: Student Name — DD-MM-YYYY [HIGH]`
  - Builds a complete HTML email body with key attendance metrics
- **Key expressions or variables used:**
  - `$input.item.json`
  - Uses fields generated by **Alert Logic Block**
  - Outputs:
    - `emailSubject`
    - `emailBody`
- **Input and output connections:**
  - Input from **Alert Logic Block**
  - Output to **Send Email**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - HTML content is directly interpolated; unusual characters in source data may affect rendering
  - Missing recipient email still allows payload generation but will break in mail delivery
- **Sub-workflow reference:** none

---

## 2.4 Parent Notification Delivery

### Overview
This block sends the generated absence emails to parents through SMTP.

### Nodes Involved
- Send Email

### Node Details

#### Send Email
- **Type and technical role:** `n8n-nodes-base.emailSend`; SMTP email delivery node.
- **Configuration choices:**
  - To: `={{ $json.email }}`
  - Subject: `={{ $json.emailSubject }}`
  - From: `={{ $vars.smtp_user }}`
  - Uses configured SMTP credentials in the node
- **Key expressions or variables used:**
  - `={{ $json.email }}`
  - `={{ $json.emailSubject }}`
  - `={{ $vars.smtp_user }}`
- **Input and output connections:**
  - Input from **Generate Absence Email**
  - No downstream connection
- **Version-specific requirements:**
  - Type version `2.1`
  - Requires SMTP credentials configured in n8n
  - Requires `smtp_user` variable defined in n8n variables
- **Edge cases or potential failure types:**
  - Missing or invalid SMTP credentials
  - Missing `smtp_user` variable
  - Empty or malformed recipient email address
  - SMTP rate limits or provider rejection
  - HTML body may not be sent unless configured in the node UI as HTML content; the JSON indicates subject/to/from, but body handling depends on the node’s content field configuration in n8n
- **Sub-workflow reference:** none

---

## 2.5 No-Absence Fallback Branch

### Overview
This branch handles executions where the IF node sends items to the false path. It creates a standard payload indicating there are no absences today so the dashboard/report generation can still proceed.

### Nodes Involved
- No Absences Today

### Node Details

#### No Absences Today
- **Type and technical role:** `n8n-nodes-base.code`; emits a fallback reporting object.
- **Configuration choices:**
  - Uses `Asia/Kolkata` timezone for date consistency
  - Produces:
    - `todayRows: []`
    - fixed CSV `headers`
    - `nowStr`
    - `today`
    - `allAbsent: []`
    - `noAbsencesToday: true`
- **Key expressions or variables used:**
  - JavaScript date formatting with `toLocaleString`
- **Input and output connections:**
  - Input from **IF: Is Absent?** false branch
  - Output to **Existing Report History**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Because the IF node evaluates per item, this branch can run whenever any today row is not absent, not only when all students are present. That can create logical ambiguity if both branches execute on the same run.
- **Sub-workflow reference:** none

---

## 2.6 Dashboard and Historical Report Update

### Overview
This block creates today’s report payload, loads existing history, merges old and new data, renders the complete dashboard HTML, converts outputs to files, and writes them back to the local file system.

### Nodes Involved
- Build Dashboard and Report
- Existing Report History
- Extract History CSV
- Combine History + Build Dashboard
- Convert Data
- Build Visual Report and Update Attendance File

### Node Details

#### Build Dashboard and Report
- **Type and technical role:** `n8n-nodes-base.code`; converts absent-student summaries into CSV row strings and report metadata.
- **Configuration choices:**
  - Collects all items from **Alert Logic Block**
  - Builds:
    - `todayRows`
    - `headers`
    - `nowStr`
    - `today`
    - `allAbsent`
    - `noAbsencesToday`
  - Marks `EmailSent` as `YES` if an email address exists, not based on actual SMTP success
  - Marks `WhatsAppSent` as `YES` if a phone exists, despite no WhatsApp node existing
- **Key expressions or variables used:**
  - `$input.all()`
- **Input and output connections:**
  - Input from **Alert Logic Block**
  - Output to **Existing Report History**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - CSV field quoting is partial; only `alertReason` is wrapped in quotes
  - “EmailSent” is inferred, not confirmed
  - “WhatsAppSent” is inferred from phone presence, not an actual send event
- **Sub-workflow reference:** none

#### Existing Report History
- **Type and technical role:** `n8n-nodes-base.readWriteFile`; reads the historical report CSV.
- **Configuration choices:**
  - Reads from `/home/node/.n8n-files/File_name.csv`
- **Key expressions or variables used:** none.
- **Input and output connections:**
  - Input from **Build Dashboard and Report** or **No Absences Today**
  - Output to **Extract History CSV**
- **Version-specific requirements:**
  - Type version `1`
- **Edge cases or potential failure types:**
  - Placeholder file path must be replaced with the actual report history file path
  - File must exist, even if empty, according to the sticky note instructions
  - Malformed CSV history can break downstream parsing
- **Sub-workflow reference:** none

#### Extract History CSV
- **Type and technical role:** `n8n-nodes-base.spreadsheetFile`; parses the existing history CSV into JSON.
- **Configuration choices:**
  - Header row enabled
- **Key expressions or variables used:** none.
- **Input and output connections:**
  - Input from **Existing Report History**
  - Output to **Combine History + Build Dashboard**
- **Version-specific requirements:**
  - Type version `2`
- **Edge cases or potential failure types:**
  - Empty file without valid header row may parse unexpectedly
  - Header mismatch with expected report schema will remove rows during filtering
- **Sub-workflow reference:** none

#### Combine History + Build Dashboard
- **Type and technical role:** `n8n-nodes-base.code`; central reporting engine that merges historical rows with today’s rows and generates two outputs: updated CSV content and dashboard HTML.
- **Configuration choices:**
  - Attempts to read built payload from:
    1. `$('Build Dashboard and Report').first().json`
    2. fallback `$('No Absences Today').first().json`
    3. final internal default object
  - Removes any existing rows for today from historical data
  - Rebuilds full CSV
  - Computes:
    - date list
    - month list
    - week list
    - class list
    - subject list
    - per-student summary stats
  - Generates a full HTML dashboard with 5 tabs:
    - Today
    - Weekly
    - Monthly
    - Full History
    - At-Risk
  - Includes Chart.js via CDN:
    - `https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.0/chart.umd.min.js`
  - Returns two items:
    - CSV output
    - HTML output
- **Key expressions or variables used:**
  - `$('Build Dashboard and Report').first().json`
  - `$('No Absences Today').first().json`
  - `$input.all()`
  - Output fields:
    - `type`
    - `textContent`
    - `fileName`
- **Input and output connections:**
  - Input from **Extract History CSV**
  - Output to **Convert Data**
- **Version-specific requirements:**
  - Type version `2`
  - Relies on Code node expression helpers referencing other nodes by name
- **Edge cases or potential failure types:**
  - If node names are changed, the internal references break
  - Large history files may produce a very large HTML string and higher memory usage
  - Dashboard depends on external Chart.js CDN; offline viewing without internet may reduce functionality
  - CSV parsing helper is simplistic and may fail on complex quoted fields
  - If history lacks expected fields such as `AlertLevel`, rows are filtered out
- **Sub-workflow reference:** none

#### Convert Data
- **Type and technical role:** `n8n-nodes-base.convertToFile`; converts text content into binary file data.
- **Configuration choices:**
  - Operation: `toText`
  - Source property: `textContent`
  - Output filename from `={{ $json.fileName }}`
- **Key expressions or variables used:**
  - `={{ $json.fileName }}`
- **Input and output connections:**
  - Input from **Combine History + Build Dashboard**
  - Output to **Build Visual Report and Update Attendance File**
- **Version-specific requirements:**
  - Type version `1`
- **Edge cases or potential failure types:**
  - Missing `textContent`
  - Filename containing invalid path characters
  - Binary mode and storage settings must permit downstream file writing
- **Sub-workflow reference:** none

#### Build Visual Report and Update Attendance File
- **Type and technical role:** `n8n-nodes-base.readWriteFile`; writes the binary file outputs to disk.
- **Configuration choices:**
  - Operation: `write`
  - Target path: `=/home/node/.n8n-files/{{ $binary.data.fileName }}`
- **Key expressions or variables used:**
  - `{{ $binary.data.fileName }}`
- **Input and output connections:**
  - Input from **Convert Data**
  - No downstream output
- **Version-specific requirements:**
  - Type version `1`
- **Edge cases or potential failure types:**
  - Potential path duplication bug: upstream `fileName` already contains an absolute path such as `/home/node/.n8n-files/attendance_report.csv`, while this node prepends `/home/node/.n8n-files/` again. This can yield an invalid target path like `/home/node/.n8n-files//home/node/.n8n-files/attendance_report.csv`.
  - File write permissions
  - Existing file overwrite behavior
- **Sub-workflow reference:** none

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Sticky Note | stickyNote | General workflow explanation and setup instructions |  |  | ## How it works<br>1. Runs every weekday at 17:30 (Mon–Fri)<br>2. Reads `student_attendance.csv` and filters to today's records only<br>3. Splits students into Absent / Present branches<br>4. Fetches parent contacts from `student_contacts.csv` and merges the data<br>5. Calculates risk level, attendance %, consecutive streak, and trend<br>6. Sends a colour-coded HTML email alert to parents of absent students via SMTP<br>7. Builds and saves an interactive `dashboard.html` with 5 tabs: Today, Weekly, Monthly, Full History, At-Risk<br>8. Appends today's records to `attendance_report.csv` for historical tracking<br><br>## Setup steps<br>1. Upload `student_attendance.csv` to the n8n files folder<br>2. Upload `student_contacts.csv` to the n8n files folder<br>3. Create an empty `attendance_report.csv` in the n8n files folder<br>4. Add `smtp_user` in n8n → Settings → Variables<br>5. Configure SMTP credentials in the **Send Email** node<br>6. Adjust the cron schedule if needed (default: 17:30 Mon–Fri) |
| Group Trigger Ingestion | stickyNote | Comment grouping trigger and input nodes |  |  | ## Trigger & data ingestion<br><br>Schedule trigger reads attendance CSV and filters to today's records only. |
| Group Routing Merge | stickyNote | Comment grouping routing and merge nodes |  |  | ## Absence routing & contact merge<br><br>Splits absent vs present students, fetches parent contacts, and merges records. |
| Group Alert Logic | stickyNote | Comment grouping alert logic nodes |  |  | ## Alert logic<br><br>Calculates risk level, attendance %, streak, and trend per student. |
| Group Notifications | stickyNote | Comment grouping notification nodes |  |  | ## Parent notifications<br><br>Generates and sends a colour-coded HTML email alert to parents via SMTP. |
| Group No Absence | stickyNote | Comment grouping no-absence branch |  |  | ## No absences branch<br><br>Skips email and dashboard when all students are present today. |
| No Absences Today | code | Emits fallback report payload for no-absence scenario | IF: Is Absent? | Existing Report History | ## No absences branch<br><br>Skips email and dashboard when all students are present today. |
| Recurring Time Trigger | scheduleTrigger | Scheduled workflow start |  | Student Attendance Data | ## Trigger & data ingestion<br><br>Schedule trigger reads attendance CSV and filters to today's records only. |
| Student Attendance Data | readWriteFile | Reads attendance CSV from disk | Recurring Time Trigger | Extract Attendance CSV | ## Trigger & data ingestion<br><br>Schedule trigger reads attendance CSV and filters to today's records only. |
| Extract Attendance CSV | spreadsheetFile | Parses attendance CSV into JSON rows | Student Attendance Data | Date Filter | ## Trigger & data ingestion<br><br>Schedule trigger reads attendance CSV and filters to today's records only. |
| IF: Is Absent? | if | Routes today’s rows by absence status | Date Filter | Students Contacts; Merge Absent + Contacts; No Absences Today | ## Absence routing & contact merge<br><br>Splits absent vs present students, fetches parent contacts, and merges records. |
| Students Contacts | readWriteFile | Reads contacts CSV from disk | IF: Is Absent? | Extract Contacts CSV | ## Absence routing & contact merge<br><br>Splits absent vs present students, fetches parent contacts, and merges records. |
| Extract Contacts CSV | spreadsheetFile | Parses contacts CSV into JSON rows | Students Contacts | Merge Absent + Contacts | ## Absence routing & contact merge<br><br>Splits absent vs present students, fetches parent contacts, and merges records. |
| Merge Absent + Contacts | merge | Joins absent attendance rows with parent contacts | IF: Is Absent?; Extract Contacts CSV | Alert Logic Block | ## Absence routing & contact merge<br><br>Splits absent vs present students, fetches parent contacts, and merges records. |
| Alert Logic Block | code | Computes per-student risk metrics and alert severity | Merge Absent + Contacts | Generate Absence Email; Build Dashboard and Report | ## Alert logic<br><br>Calculates risk level, attendance %, streak, and trend per student. |
| Generate Absence Email | code | Builds HTML email subject and body | Alert Logic Block | Send Email | ## Parent notifications<br><br>Generates and sends a colour-coded HTML email alert to parents via SMTP. |
| Build Dashboard and Report | code | Builds today’s report rows and metadata | Alert Logic Block | Existing Report History | ## Dashboard & report update<br><br>Builds interactive HTML dashboard and appends today's records to attendance history CSV. |
| Send Email | emailSend | Sends SMTP email to parent | Generate Absence Email |  | ## Parent notifications<br><br>Generates and sends a colour-coded HTML email alert to parents via SMTP. |
| Existing Report History | readWriteFile | Reads historical report CSV | Build Dashboard and Report; No Absences Today | Extract History CSV | ## Dashboard & report update<br><br>Builds interactive HTML dashboard and appends today's records to attendance history CSV. |
| Extract History CSV | spreadsheetFile | Parses historical report CSV | Existing Report History | Combine History + Build Dashboard | ## Dashboard & report update<br><br>Builds interactive HTML dashboard and appends today's records to attendance history CSV. |
| Combine History + Build Dashboard | code | Merges historical and current data, generates CSV and HTML dashboard | Extract History CSV | Convert Data | ## Dashboard & report update<br><br>Builds interactive HTML dashboard and appends today's records to attendance history CSV. |
| Convert Data | convertToFile | Converts text outputs into binary files | Combine History + Build Dashboard | Build Visual Report and Update Attendance File | ## Dashboard & report update<br><br>Builds interactive HTML dashboard and appends today's records to attendance history CSV. |
| Build Visual Report and Update Attendance File | readWriteFile | Writes CSV and HTML files to disk | Convert Data |  | ## Dashboard & report update<br><br>Builds interactive HTML dashboard and appends today's records to attendance history CSV. |
| Date Filter | filter | Keeps only today’s attendance rows | Extract Attendance CSV | IF: Is Absent? | ## Trigger & data ingestion<br><br>Schedule trigger reads attendance CSV and filters to today's records only. |
| Group Dashboard Report | stickyNote | Comment grouping dashboard/report nodes |  |  | ## Dashboard & report update<br><br>Builds interactive HTML dashboard and appends today's records to attendance history CSV. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it something like: `Track student attendance from CSV, alert parents by email, and generate an HTML dashboard`.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Name: `Recurring Time Trigger`
   - Configure cron mode with:
     - `30 17 * * 1-5`
   - This runs at 17:30 Monday to Friday.

3. **Add a file-read node for the attendance CSV**
   - Node type: **Read/Write Files from Disk**
   - Name: `Student Attendance Data`
   - Operation: read
   - File path: set to your attendance file, for example:
     - `/home/node/.n8n-files/student_attendance.csv`
   - Connect:
     - `Recurring Time Trigger -> Student Attendance Data`

4. **Parse the attendance CSV**
   - Node type: **Spreadsheet File**
   - Name: `Extract Attendance CSV`
   - Operation: read spreadsheet from binary
   - Enable:
     - `Header Row`
     - `Raw Data`
   - Connect:
     - `Student Attendance Data -> Extract Attendance CSV`

5. **Filter to today’s date**
   - Node type: **Filter**
   - Name: `Date Filter`
   - Add one condition:
     - Left value: `{{ $json.Date }}`
     - Operator: equals
     - Right value: `{{ $now.toFormat('dd-MM-yyyy') }}`
   - Connect:
     - `Extract Attendance CSV -> Date Filter`
   - Important:
     - Your CSV `Date` column must match `dd-MM-yyyy`.
     - If your source file uses another date format, add normalization before this node.

6. **Split absent vs non-absent**
   - Node type: **IF**
   - Name: `IF: Is Absent?`
   - Condition:
     - Left value: `{{ $json.Status }}`
     - Operator: equals
     - Right value: `Absent`
   - Connect:
     - `Date Filter -> IF: Is Absent?`

7. **Create the no-absence fallback code node**
   - Node type: **Code**
   - Name: `No Absences Today`
   - Paste logic that returns one JSON item with:
     - `todayRows: []`
     - `headers`
     - `nowStr`
     - `today`
     - `allAbsent: []`
     - `noAbsencesToday: true`
   - Use Asia/Kolkata formatting if you want to preserve the original behavior.
   - Connect:
     - `IF: Is Absent?` false output -> `No Absences Today`

8. **Add a file-read node for student contacts**
   - Node type: **Read/Write Files from Disk**
   - Name: `Students Contacts`
   - Operation: read
   - File path:
     - `/home/node/.n8n-files/student_contacts.csv`
   - Enable `Execute Once`
   - Connect:
     - `IF: Is Absent?` true output -> `Students Contacts`

9. **Parse the contacts CSV**
   - Node type: **Spreadsheet File**
   - Name: `Extract Contacts CSV`
   - Enable:
     - `Header Row`
   - Connect:
     - `Students Contacts -> Extract Contacts CSV`

10. **Merge absent rows with contact rows**
    - Node type: **Merge**
    - Name: `Merge Absent + Contacts`
    - Mode: **Combine**
    - Match fields:
      - Input 1 / attendance field: `StudentID`
      - Input 2 / contacts field: `StudentID`
    - Connect:
      - `IF: Is Absent?` true output -> `Merge Absent + Contacts` input 0
      - `Extract Contacts CSV -> Merge Absent + Contacts` input 1

11. **Add the attendance risk calculation node**
    - Node type: **Code**
    - Name: `Alert Logic Block`
    - Paste code that:
      - Normalizes dates
      - Groups rows by `StudentID`
      - Calculates:
        - absences in last 30 days
        - attendance percentage
        - consecutive streak
        - trend
        - alert level
        - alert reason
      - Outputs one item per student with fields such as:
        - `studentId`
        - `name`
        - `parentName`
        - `email`
        - `phone`
        - `class`
        - `subject`
        - `teacher`
        - `date`
        - `absencesLast30Days`
        - `consecutiveStreak`
        - `attendancePct`
        - `trend`
        - `alertLevel`
        - `alertReason`
    - Connect:
      - `Merge Absent + Contacts -> Alert Logic Block`

12. **Build the HTML email payload**
    - Node type: **Code**
    - Name: `Generate Absence Email`
    - Paste code that:
      - Reads one student item
      - Maps alert levels to colors and labels
      - Produces:
        - `emailSubject`
        - `emailBody`
    - Connect:
      - `Alert Logic Block -> Generate Absence Email`

13. **Configure the SMTP email node**
    - Node type: **Send Email**
    - Name: `Send Email`
    - Configure credentials:
      - Add your SMTP host, port, username, password, and security mode
    - Set fields:
      - To: `{{ $json.email }}`
      - Subject: `{{ $json.emailSubject }}`
      - From: `{{ $vars.smtp_user }}`
      - Email format/body: set the HTML body using `{{ $json.emailBody }}`
    - Connect:
      - `Generate Absence Email -> Send Email`
    - Required environment/config:
      - In n8n, create a variable named `smtp_user`
      - Set it to the sender email address

14. **Add the current-day report builder**
    - Node type: **Code**
    - Name: `Build Dashboard and Report`
    - Paste code that:
      - Collects all absent students from `$input.all()`
      - Creates CSV line strings for today’s absences
      - Produces:
        - `todayRows`
        - `headers`
        - `nowStr`
        - `today`
        - `allAbsent`
        - `noAbsencesToday`
    - Connect:
      - `Alert Logic Block -> Build Dashboard and Report`

15. **Add a file-read node for historical report CSV**
    - Node type: **Read/Write Files from Disk**
    - Name: `Existing Report History`
    - Operation: read
    - File path:
      - `/home/node/.n8n-files/attendance_report.csv`
    - Connect both:
      - `Build Dashboard and Report -> Existing Report History`
      - `No Absences Today -> Existing Report History`

16. **Create the history CSV file before first run**
    - In the n8n files directory, create:
      - `/home/node/.n8n-files/attendance_report.csv`
    - Ideally initialize it with the expected header row:
      - `Date,StudentID,Name,Class,Subject,Teacher,Status,AbsencesLast30Days,ConsecutiveStreak,AttendancePct,Trend,AlertLevel,AlertReason,ParentName,EmailSent,WhatsAppSent`

17. **Parse the historical CSV**
    - Node type: **Spreadsheet File**
    - Name: `Extract History CSV`
    - Enable:
      - `Header Row`
    - Connect:
      - `Existing Report History -> Extract History CSV`

18. **Add the dashboard generation code node**
    - Node type: **Code**
    - Name: `Combine History + Build Dashboard`
    - Paste code that:
      - Reads current payload from `Build Dashboard and Report`, or fallback from `No Absences Today`
      - Reads historical rows from `$input.all()`
      - Removes duplicate rows for today
      - Rebuilds the cumulative CSV
      - Generates the HTML dashboard
      - Returns two items:
        1. CSV content for `attendance_report.csv`
        2. HTML content for `dashboard.html`
    - Ensure the returned JSON includes:
      - `textContent`
      - `fileName`
    - Connect:
      - `Extract History CSV -> Combine History + Build Dashboard`

19. **Convert generated text to binary files**
    - Node type: **Convert to File**
    - Name: `Convert Data`
    - Operation: `toText`
    - Source property: `textContent`
    - Filename: `{{ $json.fileName }}`
    - Connect:
      - `Combine History + Build Dashboard -> Convert Data`

20. **Write the generated files to disk**
    - Node type: **Read/Write Files from Disk**
    - Name: `Build Visual Report and Update Attendance File`
    - Operation: write
    - Preferred setup:
      - Write from binary property `data`
      - Use a clean file path expression
   - Recommended correction:
      - If `fileName` already contains the full absolute path, use that directly.
      - Do not prepend `/home/node/.n8n-files/` again.
   - Example safe target:
      - `{{ $binary.data.fileName }}`
      - or return only base filenames from the previous node and prepend once here
    - Connect:
      - `Convert Data -> Build Visual Report and Update Attendance File`

21. **Add optional sticky notes**
    - Add sticky notes to document:
      - Trigger & data ingestion
      - Absence routing & contact merge
      - Alert logic
      - Parent notifications
      - No absences branch
      - Dashboard & report update
      - General setup steps

22. **Prepare required files**
    - Attendance file:
      - `student_attendance.csv`
      - Must include at least:
        - `Date`
        - `StudentID`
        - `Name`
        - `Class`
        - `Subject`
        - `Teacher`
        - `Status`
    - Contacts file:
      - `student_contacts.csv`
      - Must include at least:
        - `StudentID`
        - `ParentName`
        - `Email`
        - optionally `Phone`

23. **Test with sample data**
    - Include today’s date in `dd-MM-yyyy`
    - Include at least one `Absent` row
    - Verify:
      - Merge on `StudentID`
      - Email preview/body
      - Report CSV regeneration
      - Dashboard HTML creation

24. **Validate key implementation caveats**
    - Replace all placeholder paths like `/home/node/.n8n-files/File_name.csv`
    - Ensure SMTP body is explicitly mapped to HTML content in the email node
    - Confirm whether your IF logic should mean:
      - “this row is absent”, or
      - “today has at least one absent student”
    - If you want one dashboard run per day rather than mixed true/false branch behavior, consider aggregating rows before the IF node
    - Normalize dates before filtering if the source CSV is inconsistent

25. **Activate the workflow**
    - Once file paths, credentials, variables, and sample tests pass, activate the workflow.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow expects three files in the n8n files directory: attendance source CSV, contacts CSV, and historical report CSV. | Local file system setup |
| The intended sender address is provided through an n8n variable named `smtp_user`. | n8n Settings → Variables |
| The HTML dashboard uses Chart.js loaded from a CDN. Internet access may be required for charts to render fully when opening the saved HTML file. | `https://cdnjs.cloudflare.com/ajax/libs/Chart.js/4.4.0/chart.umd.min.js` |
| The workflow uses Asia/Kolkata explicitly inside several Code nodes, but the Schedule Trigger and Date Filter may still depend on server/n8n timezone behavior. | Timezone consistency consideration |
| The current template contains placeholder file paths and likely requires manual correction before production use. | Replace `/home/node/.n8n-files/File_name.csv` with real filenames |
| The final file-writing node likely needs path correction to avoid prepending the base folder twice. | Review write-path expression before use |