Send multi-stage customer project emails from Google Sheets with Zoho ZeptoMail

https://n8nworkflows.xyz/workflows/send-multi-stage-customer-project-emails-from-google-sheets-with-zoho-zeptomail-14994


# Send multi-stage customer project emails from Google Sheets with Zoho ZeptoMail

The user wants me to analyze the given n8n workflow JSON and create a comprehensive reference document in English, following the exact structure specified. Let me parse through the workflow JSON carefully and document everything.

Let me break down the workflow:

**Nodes:**

1. Main Sticky - sticky note
2. Daily Schedule Trigger - scheduleTrigger
3. Read All Customer Rows - googleSheets (read)
4. Filter Empty Rows - filter
5. Calculate Date Variables - set
6. Is T+3 Email Due? - if
7. Loop T+3 (1 at a time) - splitInBatches
8. Wait (T+3) - wait
9. Mark T+3 Email Sent - googleSheets (update)
10. Send T+3 Update Email - zohoZeptomail
11. Is T+5 Email Due? - if
12. Loop T+5 (1 at a time) - splitInBatches
13. Wait (T+5) - wait
14. Mark T+5 Email Sent - googleSheets (update)
15. Send T+5 Update Email - zohoZeptomail
16. Is Feedback Email Due? - if
17. Loop Feedback (1 at a time) - splitInBatches
18. Wait (Feedback) - wait
19. Mark Feedback Email Sent - googleSheets (update)
20. Send Feedback Request Email - zohoZeptomail
21. Is Upsell Email Due? - if
22. Loop Upsell (1 at a time) - splitInBatches
23. Wait (Upsell) - wait
24. Mark Upsell Email Sent - googleSheets (update)
25. Send Upsell Email1 - zohoZeptomail
26. Section 1 - sticky note
27. Section 2 - sticky note
28. Section 3 - sticky note

**Connections:**
- Daily Schedule Trigger → Read All Customer Rows
- Read All Customer Rows → Filter Empty Rows
- Filter Empty Rows → Calculate Date Variables
- Calculate Date Variables → Is T+3 Email Due?, Is T+5 Email Due?, Is Feedback Email Due?, Is Upsell Email Due?
- Is T+3 Email Due? (true) → Loop T+3 (1 at a time)
- Loop T+3 (1 at a time) (batch done → Wait T+3)
- Wait (T+3) → Send T+3 Update Email
- Send T+3 Update Email → Mark T+3 Email Sent
- Mark T+3 Email Sent → Loop T+3 (1 at a time) (loop back)
- Is T+5 Email Due? (true) → Loop T+5 (1 at a time)
- Loop T+5 (1 at a time) (batch done → Wait T+5)
- Wait (T+5) → Send T+5 Update Email
- Send T+5 Update Email → Mark T+5 Email Sent
- Mark T+5 Email Sent → Loop T+5 (1 at a time) (loop back)
- Is Feedback Email Due? (true) → Loop Feedback (1 at a time)
- Loop Feedback (1 at a time) (batch done → Wait Feedback)
- Wait (Feedback) → Send Feedback Request Email
- Send Feedback Request Email → Mark Feedback Email Sent
- Mark Feedback Email Sent → Loop Feedback (1 at a time) (loop back)
- Is Upsell Email Due? (true) → Loop Upsell (1 at a time)
- Loop Upsell (1 at a time) (batch done → Wait Upsell)
- Wait (Upsell) → Send Upsell Email1
- Send Upsell Email1 → Mark Upsell Email Sent
- Mark Upsell Email Sent → Loop Upsell (1 at a time) (loop back)

Now let me structure the document.

**Logical blocks:**

1. **Trigger & Data Retrieval** - Daily Schedule Trigger, Read All Customer Rows, Filter Empty Rows
2. **Date Calculation & Branching** - Calculate Date Variables, four IF nodes
3. **T+3 Email Branch** - Loop T+3, Wait T+3, Send T+3 Update Email, Mark T+3 Email Sent
4. **T+5 Email Branch** - Loop T+5, Wait T+5, Send T+5 Update Email, Mark T+5 Email Sent
5. **Feedback Email Branch** - Loop Feedback, Wait Feedback, Send Feedback Request Email, Mark Feedback Email Sent
6. **Upsell Email Branch** - Loop Upsell, Wait Upsell, Send Upsell Email1, Mark Upsell Email Sent

Now let me detail each node.

Let me analyze each node's configuration in depth.

**Daily Schedule Trigger:**
- Type: scheduleTrigger v1.2
- Config: interval rule, triggerAtHour: 9 (9 AM)
- Note: "Runs daily at 9:00 AM IST."
- Output: triggers Read All Customer Rows

**Read All Customer Rows:**
- Type: googleSheets v4.4
- Operation: read (default)
- sheetName and documentId: both set to YOUR_RESOURCE_ID_HERE (mode: id)
- Options: empty
- Output: all rows from the sheet, feeds into Filter Empty Rows

**Filter Empty Rows:**
- Type: filter v2
- Conditions: AND - Email not equal to empty string, Customer Name not equal to empty string
- Case-sensitive strict validation
- Note: "Skips blank rows. Only rows with a name + email pass through."
- Output: filtered rows → Calculate Date Variables

**Calculate Date Variables:**
- Type: set v3.4
- Include other fields: true
- Assignments:
  - today: $now.toFormat('yyyy-MM-dd')
  - t3_target_date: $now.minus({ days: 3 }).toFormat('yyyy-MM-dd')
  - t3_window_date: $now.minus({ days: 7 }).toFormat('yyyy-MM-dd')
  - t5_target_date: $now.minus({ days: 5 }).toFormat('yyyy-MM-dd')
  - t5_window_date: $now.minus({ days: 10 }).toFormat('yyyy-MM-dd')
  - upsell_target_date: $now.minus({ days: 3 }).toFormat('yyyy-MM-dd')
  - upsell_window_date: $now.minus({ days: 7 }).toFormat('yyyy-MM-dd')
- Note: "FIX: t5_window_date is now 10 days (was 9) to catch boundary customers."
- Output: passes enriched items to all four IF nodes

**Is T+3 Email Due?**
- Type: if v2.1
- Loose type validation: true
- Conditions AND:
  - Date Added after t3_window_date
  - Date Added before or equals t3_target_date
  - T3_Email_Sent is empty
  - Status not equal to Completed
  - Status not equal to All Done
- True branch → Loop T+3 (1 at a time)

**Loop T+3 (1 at a time)**
- Type: splitInBatches v3
- Processes items one at a time
- Output 0 (done): loops back after marking sent
- Output 1 (each item): → Wait (T+3)

**Wait (T+3)**
- Type: wait v1.1
- Amount: 20 (seconds? minutes? - default is seconds I believe, so 20 seconds pause)
- Output → Send T+3 Update Email

**Send T+3 Update Email**
- Type: n8n-nodes-zohozeptomail.zohoZeptomail v1
- Parameters:
  - subject: dynamic: 'Update on Your ' + Project Type + ' Project - We\'re Making Progress!'
  - htmlbody: HTML email with Day 3 update template
  - mailagent: '555046c28339d27e'
  - toaddress: $json['Email']
  - fromaddress: { name: "Think Sage", address: "YOUR_EMAIL_HERE" }
- Output → Mark T+3 Email Sent

**Mark T+3 Email Sent**
- Type: googleSheets v4.4
- Operation: update
- sheetName/documentId: YOUR_RESOURCE_ID_HERE
- matchingColumns: row_number
- Columns mapped with expressions referencing Loop T+3 (1 at a time).item.json for original values, and setting:
  - T3_Email_Sent = "YES"
  - original_subject = 'Update on Your ' + Project Type + ' Project - We\'re Making Progress!'
- All other columns preserved from loop item
- Output → loops back to Loop T+3 (1 at a time)

**Is T+5 Email Due?**
- Type: if v2.1
- Conditions AND (strict):
  - Date Added after t5_window_date
  - Date Added before or equals t5_target_date
  - T5_Email_Sent is empty
  - Status not equal to Completed
  - Status not equal to All Done
  - T3_Email_Sent equals YES
- Note: "FIX: Requires T3_Email_Sent=YES so T+5 never sends before or at the same time as T+3."
- True branch → Loop T+5 (1 at a time)

**Loop T+5 (1 at a time)**
- splitInBatches v3
- Output 1 → Wait (T+5)

**Wait (T+5)**
- wait v1.1, amount 20

**Send T+5 Update Email**
- zohoZeptomail v1
- subject: $json['original_subject'] (preserves subject from T+3)
- htmlbody: Day 5 update template
- mailagent: 555046c28339d27e
- fromaddress: Think Sage / YOUR_EMAIL_HERE
- toaddress: $json['Email']

**Mark T+5 Email Sent**
- googleSheets update
- matchingColumns: row_number
- Sets T5_Email_Sent = "YES"
- Preserves original_subject from loop item
- All other fields preserved
- Output → Loop T+5 (1 at a time) (loop back)

**Is Feedback Email Due?**
- if v2.1
- Conditions AND (strict):
  - Status not equal to "In Progress"
  - Status not equal to "All Done"
  - Feedback_Email_Sent is empty
  - T3_Email_Sent equals YES
  - T5_Email_Sent equals YES
- Note: "FIX: Added All Done check so completed+upselled customers don't get feedback emails again."
- True → Loop Feedback (1 at a time)

**Loop Feedback (1 at a time)**
- splitInBatches v3

**Wait (Feedback)**
- wait v1.1, amount 20

**Send Feedback Request Email**
- zohoZeptomail v1
- subject: $json['original_subject']
- htmlbody: feedback request template with link placeholder YOUR_FEEDBACK_FORM_LINK
- fromaddress: Think Sage / YOUR_EMAIL_HERE
- toaddress: $json['Email']

**Mark Feedback Email Sent**
- googleSheets update
- matchingColumns: row_number
- Sets Feedback_Email_Sent = "YES"
- Sets Feedback_Email_Date = $now.toFormat('yyyy-MM-dd')
- All other fields preserved from loop item
- Output → Loop Feedback (1 at a time) (loop back)

**Is Upsell Email Due?**
- if v2.1
- Conditions AND (strict):
  - Status not equal to "In Progress"
  - Status not equal to "All Done"
  - Feedback_Email_Sent equals YES
  - Feedback_Email_Date after upsell_window_date
  - Feedback_Email_Date before or equals upsell_target_date
  - Upsell_Email_Sent is empty
- True → Loop Upsell (1 at a time)

**Loop Upsell (1 at a time)**
- splitInBatches v3

**Wait (Upsell)**
- wait v1.1, amount 20

**Send Upsell Email1**
- zohoZeptomail v1
- subject: $json['original_subject']
- htmlbody: upsell email template with booking link placeholder YOUR_BOOKING_LINK
- fromaddress: Think Sage / YOUR_EMAIL_HERE
- toaddress: $json['Email']

**Mark Upsell Email Sent**
- googleSheets update
- matchingColumns: row_number
- Sets Upsell_Email_Sent = "YES"
- Sets Status = "All Done"
- Note: "Sets Status = All Done after upsell -- stops all future checks for this customer."
- All other fields preserved from loop item
- Output → Loop Upsell (1 at a time) (loop back)

**Sticky Notes:**
- Main Sticky: color 2, content with description of workflow, setup, customization, important note about community node
- Section 1: color 7, "## 1. Trigger & Data Retrieval - Polls the sheet daily and filters out blank rows."
- Section 2: color 7, "## 2. Logic & Scheduling - Calculates dynamic dates and checks email due status."
- Section 3: color 7, "## 3. Communication & Logging - Sends emails via ZeptoMail and updates Google Sheets."

Now let me organize the document.

Section 1: Workflow Overview
- Purpose, use cases, logical blocks

Section 2: Block-by-Block Analysis
- Block 1: Trigger & Data Retrieval
- Block 2: Date Calculation & Branching
- Block 3: T+3 Email Branch
- Block 4: T+5 Email Branch
- Block 5: Feedback Email Branch
- Block 6: Upsell Email Branch

Each block with overview, nodes involved, node details.

Section 3: Summary Table

Section 4: Reproducing the Workflow from Scratch

Section 5: General Notes & Resources

Let me compile all details.

I need to be thorough and not skip any node.

Let me also note edge cases and potential failure points.

Edge cases:
- Google Sheets credentials not set
- Zoho ZeptoMail credentials not set
- Community node not installed
- Date Added column format mismatch (must be parseable as dateTime)
- Empty/blank rows causing filter to pass unexpected data
- Timezone considerations (trigger at 9 AM but timezone depends on n8n server settings)
- Wait node of 20 seconds might be too short for rate limiting
- ZeptoMail mailagent ID must be valid
- Sheet column names must match exactly (case-sensitive)
- row_number used as matching column must exist
- If Status column contains values other than "In Progress", "Completed", "All Done"
- The upsell email template has placeholder links that must be replaced

Let me also consider the Wait nodes. They wait 20 (units not specified - likely seconds). This is probably to avoid rate limiting when sending multiple emails in rapid succession.

Let me also document the expressions carefully.

Now let me write the document.

I need to include sticky note content for relevant nodes. The sticky notes that cover nodes:
- Main Sticky covers the entire workflow (general instructions)
- Section 1 sticky covers: Daily Schedule Trigger, Read All Customer Rows, Filter Empty Rows
- Section 2 sticky covers: Calculate Date Variables, the four IF nodes
- Section 3 sticky covers: all Loop, Wait, Send, and Mark nodes

I'll include the relevant sticky note content in each node's row in the summary table.

Let me prepare the summary table with all 25 functional nodes + sticky notes.

Actually, I have 4 sticky note nodes (Main Sticky, Section 1, Section 2, Section 3). These are not functional nodes but should still be listed in the summary table per the instructions "Do not skip any nodes."

Let me list all nodes:

1. Main Sticky - stickyNote
2. Daily Schedule Trigger - scheduleTrigger
3. Read All Customer Rows - googleSheets
4. Filter Empty Rows - filter
5. Calculate Date Variables - set
6. Is T+3 Email Due? - if
7. Loop T+3 (1 at a time) - splitInBatches
8. Wait (T+3) - wait
9. Send T+3 Update Email - zohoZeptomail
10. Mark T+3 Email Sent - googleSheets
11. Is T+5 Email Due? - if
12. Loop T+5 (1 at a time) - splitInBatches
13. Wait (T+5) - wait
14. Send T+5 Update Email - zohoZeptomail
15. Mark T+5 Email Sent - googleSheets
16. Is Feedback Email Due? - if
17. Loop Feedback (1 at a time) - splitInBatches
18. Wait (Feedback) - wait
19. Send Feedback Request Email - zohoZeptomail
20. Mark Feedback Email Sent - googleSheets
21. Is Upsell Email Due? - if
22. Loop Upsell (1 at a time) - splitInBatches
23. Wait (Upsell) - wait
24. Send Upsell Email1 - zohoZeptomail
25. Mark Upsell Email Sent - googleSheets
26. Section 1 - stickyNote
27. Section 2 - stickyNote
28. Section 3 - stickyNote

Wait, I also need to check whether Section 1, 2, 3 sticky notes should be in the summary table. The instructions say list all workflow nodes. Sticky notes are nodes, so yes, include them.

For the Sticky Note column, I need to include any associated comment. The main sticky note has a big content. The section sticky notes have their own content. And some nodes have notes in the workflow (like Filter Empty Rows has a note, Calculate Date Variables has a note, Is T+5 Email Due? has a note, Is Feedback Email Due? has a note, Mark Upsell Email Sent has a note). These are separate from sticky notes - they're node notes. The instructions say "Sticky Note column must include any associated comment (no sticky note ID)." I think this means both sticky note content AND node notes. Let me include them.

Let me map:

- Main Sticky: its own content
- Daily Schedule Trigger: note "Runs daily at 9:00 AM IST."
- Read All Customer Rows: (covered by Section 1 sticky note)
- Filter Empty Rows: note "Skips blank rows. Only rows with a name + email pass through." + Section 1 sticky
- Calculate Date Variables: note "FIX: t5_window_date is now 10 days (was 9) to catch boundary customers." + Section 2 sticky
- Is T+3 Email Due?: Section 2 sticky
- Loop T+3 (1 at a time): Section 3 sticky
- Wait (T+3): Section 3 sticky
- Send T+3 Update Email: Section 3 sticky
- Mark T+3 Email Sent: Section 3 sticky
- Is T+5 Email Due?: note "FIX: Requires T3_Email_Sent=YES so T+5 never sends before or at the same time as T+3." + Section 2 sticky
- Loop T+5 (1 at a time): Section 3 sticky
- Wait (T+5): Section 3 sticky
- Send T+5 Update Email: Section 3 sticky
- Mark T+5 Email Sent: Section 3 sticky
- Is Feedback Email Due?: note "FIX: Added All Done check so completed+upselled customers don't get feedback emails again." + Section 2 sticky
- Loop Feedback (1 at a time): Section 3 sticky
- Wait (Feedback): Section 3 sticky
- Send Feedback Request Email: Section 3 sticky
- Mark Feedback Email Sent: Section 3 sticky
- Is Upsell Email Due?: Section 2 sticky
- Loop Upsell (1 at a time): Section 3 sticky
- Wait (Upsell): Section 3 sticky
- Send Upsell Email1: Section 3 sticky
- Mark Upsell Email Sent: note "Sets Status = All Done after upsell -- stops all future checks for this customer." + Section 3 sticky

Section 1 sticky: "## 1. Trigger & Data Retrieval\nPolls the sheet daily and filters out blank rows."
Section 2 sticky: "## 2. Logic & Scheduling\nCalculates dynamic dates and checks email due status."
Section 3 sticky: "## 3. Communication & Logging\nSends emails via ZeptoMail and updates Google Sheets."
Main Sticky: full content about automation, setup, customization, important note.

OK now let me write the full document.

I need to be very thorough in the Block-by-Block Analysis section, covering each node's configuration, expressions, connections, edge cases, etc.

Let me write it now.### 1. Workflow Overview

**Purpose**  
This workflow automates a four-stage email follow‑up sequence for service‑based businesses (resume writing, LinkedIn optimization, design, freelance agencies, etc.) that manage client orders in a Google Sheet. It runs once per day, inspects every non‑empty row, decides which email stage is due based on date arithmetic and status flags, sends the appropriate message through Zoho ZeptoMail, and writes back a “sent” marker to the same row—ensuring no email is ever sent twice. Once the final upsell email is dispatched, the row’s Status is set to **All Done**, permanently retiring it from future checks.

**Logical Blocks**

| # | Block Name | Function |
|---|------------|----------|
| 1 | Trigger & Data Retrieval | Daily schedule trigger → reads all rows from a Google Sheet → discards blank rows |
| 2 | Date Calculation & Branching | Computes dynamic target/window dates for each stage → fans out to four parallel IF conditions |
| 3 | T+3 Email Branch | Loops qualifying rows one‑by‑one, pauses briefly, sends the Day‑3 progress update, logs T3_Email_Sent = YES |
| 4 | T+5 Email Branch | Same pattern for Day‑5 near‑completion update (requires T3 confirmed), logs T5_Email_Sent = YES |
| 5 | Feedback Email Branch | Triggered when Status ≠ In Progress and both T3/T5 are sent, logs Feedback_Email_Sent = YES and records the date |
| 6 | Upsell Email Branch | Sent ~3 days after the feedback email, logs Upsell_Email_Sent = YES and sets Status = All Done |

All four branches operate independently; a single daily run can execute any combination of them.

---

### 2. Block‑by‑Block Analysis

---

#### Block 1 – Trigger & Data Retrieval

**Overview**  
A scheduled trigger fires each day at 9 AM. It pulls every row from a designated Google Sheet and then filters out rows that lack a customer name or email address, producing a clean dataset for downstream logic.

**Nodes Involved**  
- Daily Schedule Trigger  
- Read All Customer Rows  
- Filter Empty Rows  

**Node Details**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input → Output | Edge Cases / Failure Types |
|------|-------------|---------------|-----------------------------|----------------|---------------------------|
| **Daily Schedule Trigger** | `scheduleTrigger` (v1.2) – workflow starter | Runs daily at 9:00 (server timezone). Interval rule with `triggerAtHour: 9`. | – | Output triggers **Read All Customer Rows** | Wrong timezone on server → emails fire at unexpected local time |
| **Read All Customer Rows** | `googleSheets` (v4.4) – reads sheet data | Operation: read (default). Both `documentId` and `sheetName` are set via ID mode to `YOUR_RESOURCE_ID_HERE` placeholders. Options left empty. | – | Receives trigger → outputs an array of row objects (each row becomes an item) | Missing or invalid resource IDs → 404; credential not configured → auth error; sheet columns renamed → downstream expressions fail |
| **Filter Empty Rows** | `filter` (v2) – removes blank rows | Combinator: **AND**. Two strict string conditions: `Email ≠ ""` and `Customer Name ≠ ""`. Case‑sensitive, strict type validation. | `={{ $json['Email'] }}` and `={{ $json['Customer Name'] }}` | Input from **Read All Customer Rows** → true branch feeds **Calculate Date Variables** | If a row has whitespace only, it passes the filter; consider trimming in the sheet or adding regex conditions |

---

#### Block 2 – Date Calculation & Branching

**Overview**  
A Set node computes seven date variables relative to “today” that define windows for each email stage. The enriched items then fan out to four parallel IF nodes, each testing whether its respective email is due.

**Nodes Involved**  
- Calculate Date Variables  
- Is T+3 Email Due?  
- Is T+5 Email Due?  
- Is Feedback Email Due?  
- Is Upsell Email Due?  

**Node Details**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input → Output | Edge Cases / Failure Types |
|------|-------------|---------------|-----------------------------|----------------|---------------------------|
| **Calculate Date Variables** | `set` (v3.4) – adds computed date fields | `includeOtherFields: true` (preserves original columns). Seven assignments, all string type: <br>• `today` = `$now.toFormat('yyyy-MM-dd')` <br>• `t3_target_date` = `$now.minus({days:3})` <br>• `t3_window_date` = `$now.minus({days:7})` <br>• `t5_target_date` = `$now.minus({days:5})` <br>• `t5_window_date` = `$now.minus({days:10})` <br>• `upsell_target_date` = `$now.minus({days:3})` <br>• `upsell_window_date` = `$now.minus({days:7})` | All date strings formatted as `yyyy-MM-dd`. | Input from **Filter Empty Rows** → output passes to all four IF nodes simultaneously | Server timezone matters—`$now` reflects the n8n process timezone. If “Date Added” is stored in a different format, the dateTime comparisons in IF nodes will fail |
| **Is T+3 Email Due?** | `if` (v2.1) – decides T+3 eligibility | Loose type validation. Conditions (AND): <br>1. `Date Added` **after** `t3_window_date` <br>2. `Date Added` **before or equals** `t3_target_date` <br>3. `T3_Email_Sent` **empty** <br>4. `Status` **≠** `Completed` <br>5. `Status` **≠** `All Done` | Left values reference `$json['Date Added']`, `$json.t3_window_date`, etc. | Input from **Calculate Date Variables** → true branch → **Loop T+3 (1 at a time)**; false branch discarded | If “Date Added” is not a recognizable date string, the dateTime operator fails silently (item routed to false) |
| **Is T+5 Email Due?** | `if` (v2.1) – decides T+5 eligibility | Strict type validation. Conditions (AND): <br>1. `Date Added` **after** `t5_window_date` <br>2. `Date Added` **before or equals** `t5_target_date` <br>3. `T5_Email_Sent` **empty** <br>4. `Status` **≠** `Completed` <br>5. `Status` **≠** `All Done` <br>6. `T3_Email_Sent` **equals** `YES` | Same pattern; extra condition ensures T+5 never fires before T+3 is confirmed. | Input from **Calculate Date Variables** → true → **Loop T+5 (1 at a time)** | If T3_Email_Sent is misspelled or not exactly “YES”, the condition fails |
| **Is Feedback Email Due?** | `if` (v2.1) – decides feedback eligibility | Strict validation. Conditions (AND): <br>1. `Status` **≠** `In Progress` <br>2. `Status` **≠** `All Done` <br>3. `Feedback_Email_Sent` **empty** <br>4. `T3_Email_Sent` **equals** `YES` <br>5. `T5_Email_Sent` **equals** `YES` | Requires both prior emails confirmed. | Input from **Calculate Date Variables** → true → **Loop Feedback (1 at a time)** | A row where Status = “Completed” but T3/T5 are missing will be excluded |
| **Is Upsell Email Due?** | `if` (v2.1) – decides upsell eligibility | Strict validation. Conditions (AND): <br>1. `Status` **≠** `In Progress` <br>2. `Status` **≠** `All Done` <br>3. `Feedback_Email_Sent` **equals** `YES` <br>4. `Feedback_Email_Date` **after** `upsell_window_date` <br>5. `Feedback_Email_Date` **before or equals** `upsell_target_date` <br>6. `Upsell_Email_Sent` **empty** | Uses dynamic window/target dates computed earlier. | Input from **Calculate Date Variables** → true → **Loop Upsell (1 at a time)** | If Feedback_Email_Date is empty (should not happen if Feedback_Email_Sent=YES), the date comparison will fail |

---

#### Block 3 – T+3 Email Branch

**Overview**  
Qualifying rows are processed one at a time. After a brief pause (to respect rate limits), a progress‑update email is sent via Zoho ZeptoMail. The sheet is then updated with `T3_Email_Sent = YES` and the original subject line, and the loop repeats for the next item.

**Nodes Involved**  
- Loop T+3 (1 at a time)  
- Wait (T+3)  
- Send T+3 Update Email  
- Mark T+3 Email Sent  

**Node Details**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input → Output | Edge Cases / Failure Types |
|------|-------------|---------------|-----------------------------|----------------|---------------------------|
| **Loop T+3 (1 at a time)** | `splitInBatches` (v3) – processes items sequentially | Batch size = 1 (default). Output 0 (all done) loops back after marking; output 1 (each item) goes to **Wait (T+3)**. | – | Input from **Is T+3 Email Due?** (true branch) → each item to **Wait (T+3)** | If zero items qualify, the loop ends immediately |
| **Wait (T+3)** | `wait` (v1.1) – brief delay | Amount: 20 (seconds by default). | – | Input from **Loop T+3** output 1 → **Send T+3 Update Email** | Adjust if ZeptoMail rate limits require longer spacing |
| **Send T+3 Update Email** | `n8n-nodes-zohozeptomail.zohoZeptomail` (v1) – sends Day‑3 email | <br>• `subject`: `={{ 'Update on Your ' + $json['Project Type'] + ' Project - We\'re Making Progress!' }}` <br>• `htmlbody`: styled HTML with Day‑3 messaging, referencing `$json['Customer Name']` and `$json['Project Type']` <br>• `mailagent`: `555046c28339d27e` <br>• `toaddress`: `={{ $json['Email'] }}` <br>• `fromaddress`: `{ name: "Think Sage", address: "YOUR_EMAIL_HERE" }` | Dynamic subject and body built from row data. Sender address is a placeholder. | Input from **Wait (T+3)** → output → **Mark T+3 Email Sent** | Invalid mailagent ID → send failure; placeholder email not replaced → bounce; ZeptoMail auth issue |
| **Mark T+3 Email Sent** | `googleSheets` (v4.4) – writes back to sheet | Operation: update. Matching column: `row_number`. Columns mapped: all original columns from the loop item, with two overrides: <br>• `T3_Email_Sent` = `"YES"` <br>• `original_subject` = `={{ 'Update on Your ' + $('Loop T+3 (1 at a time)').item.json['Project Type'] + ' Project - We\'re Making Progress!' }}` | Uses `$('Loop T+3 (1 at a time)').item.json` to preserve original row data. | Input from **Send T+3 Update Email** → output → loops back to **Loop T+3 (1 at a time)** (to process next batch item) | If `row_number` column is missing or duplicated, update may write to wrong row |

---

#### Block 4 – T+5 Email Branch

**Overview**  
Mirrors the T+3 pattern but only fires after `T3_Email_Sent = YES`. Sends a near‑completion update, then logs `T5_Email_Sent = YES` while preserving the previously saved subject.

**Nodes Involved**  
- Loop T+5 (1 at a time)  
- Wait (T+5)  
- Send T+5 Update Email  
- Mark T+5 Email Sent  

**Node Details**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input → Output | Edge Cases / Failure Types |
|------|-------------|---------------|-----------------------------|----------------|---------------------------|
| **Loop T+5 (1 at a time)** | `splitInBatches` (v3) | Same as T+3 loop. Output 1 → **Wait (T+5)**. | – | Input from **Is T+5 Email Due?** true → each item onward | – |
| **Wait (T+5)** | `wait` (v1.1) | Amount: 20 seconds. | – | → **Send T+5 Update Email** | – |
| **Send T+5 Update Email** | `zohoZeptomail` (v1) | <br>• `subject`: `={{ $json['original_subject'] }}` <br>• `htmlbody`: Day‑5 HTML template referencing `$json['Customer Name']` and `$json['Project Type']` <br>• `mailagent`: `555046c28339d27e` <br>• `toaddress`: `={{ $json['Email'] }}` <br>• `fromaddress`: `{ name: "Think Sage", address: "YOUR_EMAIL_HERE" }` | Subject reuses the value stored in `original_subject` from the T+3 step. | → **Mark T+5 Email Sent** | Missing `original_subject` on rows that bypassed T+3 (shouldn’t happen due to IF condition) |
| **Mark T+5 Email Sent** | `googleSheets` (v4.4) | Operation: update. Matching column: `row_number`. Overrides: <br>• `T5_Email_Sent` = `"YES"` <br>• All other columns carried forward from `$('Loop T+5 (1 at a time)').item.json`. | – | → loops back to **Loop T+5 (1 at a time)** | Same row_number caveats as T+3 |

---

#### Block 5 – Feedback Email Branch

**Overview**  
Triggered when the project is no longer “In Progress” and both T3/T5 emails are confirmed. Sends a feedback request with a call‑to‑action link, logs `Feedback_Email_Sent = YES` and records today’s date in `Feedback_Email_Date`.

**Nodes Involved**  
- Loop Feedback (1 at a time)  
- Wait (Feedback)  
- Send Feedback Request Email  
- Mark Feedback Email Sent  

**Node Details**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input → Output | Edge Cases / Failure Types |
|------|-------------|---------------|-----------------------------|----------------|---------------------------|
| **Loop Feedback (1 at a time)** | `splitInBatches` (v3) | Output 1 → **Wait (Feedback)**. | – | Input from **Is Feedback Email Due?** true | – |
| **Wait (Feedback)** | `wait` (v1.1) | Amount: 20 seconds. | – | → **Send Feedback Request Email** | – |
| **Send Feedback Request Email** | `zohoZeptomail` (v1) | <br>• `subject`: `={{ $json['original_subject'] }}` <br>• `htmlbody`: HTML with feedback CTA button linking to `YOUR_FEEDBACK_FORM_LINK` placeholder <br>• `mailagent`: `555046c28339d27e` <br>• `toaddress`: `={{ $json['Email'] }}` <br>• `fromaddress`: `{ name: "Think Sage", address: "YOUR_EMAIL_HERE" }` | Placeholder link must be replaced with actual form URL. | → **Mark Feedback Email Sent** | Link not replaced → broken CTA; ZeptoMail errors |
| **Mark Feedback Email Sent** | `googleSheets` (v4.4) | Operation: update. Matching column: `row_number`. Overrides: <br>• `Feedback_Email_Sent` = `"YES"` <br>• `Feedback_Email_Date` = `={{ $now.toFormat('yyyy-MM-dd') }}` <br>• All other columns preserved from loop item. | – | → loops back to **Loop Feedback (1 at a time)** | Date format must match what the Upsell IF node expects (yyyy‑MM‑dd) |

---

#### Block 6 – Upsell Email Branch

**Overview**  
Sent approximately three days after the feedback email (window‑based check). Includes an upsell table and discount code. After sending, the row’s `Upsell_Email_Sent` is marked `YES` and the Status is set to **All Done**, which permanently stops all future processing for that customer.

**Nodes Involved**  
- Loop Upsell (1 at a time)  
- Wait (Upsell)  
- Send Upsell Email1  
- Mark Upsell Email Sent  

**Node Details**

| Node | Type & Role | Configuration | Key Expressions / Variables | Input → Output | Edge Cases / Failure Types |
|------|-------------|---------------|-----------------------------|----------------|---------------------------|
| **Loop Upsell (1 at a time)** | `splitInBatches` (v3) | Output 1 → **Wait (Upsell)**. | – | Input from **Is Upsell Email Due?** true | – |
| **Wait (Upsell)** | `wait` (v1.1) | Amount: 20 seconds. | – | → **Send Upsell Email1** | – |
| **Send Upsell Email1** | `zohoZeptomail` (v1) | <br>• `subject`: `={{ $json['original_subject'] }}` <br>• `htmlbody`: HTML upsell template with table of premium services, discount code `RETURN15`, and CTA button linking to `YOUR_BOOKING_LINK` placeholder <br>• `mailagent`: `555046c28339d27e` <br>• `toaddress`: `={{ $json['Email'] }}` <br>• `fromaddress`: `{ name: "Think Sage", address: "YOUR_EMAIL_HERE" }` | Two placeholders must be replaced: booking link and sender email. | → **Mark Upsell Email Sent** | Link not replaced → broken CTA |
| **Mark Upsell Email Sent** | `googleSheets` (v4.4) | Operation: update. Matching column: `row_number`. Overrides: <br>• `Upsell_Email_Sent` = `"YES"` <br>• `Status` = `"All Done"` <br>• All other columns preserved from loop item. | – | → loops back to **Loop Upsell (1 at a time)** (terminates when all batches processed) | Setting Status to “All Done” ensures the row is excluded from all four IF checks on subsequent daily runs |

---

#### Sticky Notes (Documentation Nodes)

| Node | Type | Content (Preserved) |
|------|------|---------------------|
| **Main Sticky** | `stickyNote` (v1) | **## Automate Customer Project Emails**<br>Streamline client updates and feedback requests with automated multi‑stage email workflows.<br><br>### How it works<br>1. Fetch daily customer data from Google Sheets.<br>2. Filter for active clients and incomplete tasks.<br>3. Calculate target dates for T+3 and T+5 intervals.<br>4. Trigger personalized emails based on project progress.<br>5. Log sent status back to Google Sheets.<br><br>### Setup<br>1. Connect Google Sheets and Zoho ZeptoMail credentials.<br>2. Configure your Resource IDs for the sheets.<br>3. Add your sender email and links in the email nodes.<br>4. Install required community nodes (see Important section).<br><br>### Customization<br>Adjust the “Calculate Date Variables” node to change the delay intervals. Consolidate user‑specific values in a Set node at the workflow start for easy configuration.<br><br>### Important<br>This workflow requires a self‑hosted n8n instance. Community nodes used: n8n‑nodes‑zohozeptomail.zohoZeptomail |
| **Section 1** | `stickyNote` (v1) | **## 1. Trigger & Data Retrieval**<br>Polls the sheet daily and filters out blank rows. |
| **Section 2** | `stickyNote` (v1) | **## 2. Logic & Scheduling**<br>Calculates dynamic dates and checks email due status. |
| **Section 3** | `stickyNote` (v1) | **## 3. Communication & Logging**<br>Sends emails via ZeptoMail and updates Google Sheets. |

---

### 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|-----------|-----------|-----------------|---------------|----------------|-------------|
| Main Sticky | stickyNote | Documentation / overview | – | – | ## Automate Customer Project Emails — Streamline client updates and feedback requests with automated multi‑stage email workflows. **How it works**: 1. Fetch daily customer data from Google Sheets. 2. Filter for active clients and incomplete tasks. 3. Calculate target dates for T+3 and T+5 intervals. 4. Trigger personalized emails based on project progress. 5. Log sent status back to Google Sheets. **Setup**: 1. Connect Google Sheets and Zoho ZeptoMail credentials. 2. Configure your Resource IDs for the sheets. 3. Add your sender email and links in the email nodes. 4. Install required community nodes (see Important section). **Customization**: Adjust the “Calculate Date Variables” node to change the delay intervals. Consolidate user‑specific values in a Set node at the workflow start for easy configuration. **Important**: This workflow requires a self‑hosted n8n instance. Community nodes used: n8n‑nodes‑zohozeptomail.zohoZeptomail |
| Section 1 | stickyNote | Section label | – | – | ## 1. Trigger & Data Retrieval — Polls the sheet daily and filters out blank rows. |
| Section 2 | stickyNote | Section label | – | – | ## 2. Logic & Scheduling — Calculates dynamic dates and checks email due status. |
| Section 3 | stickyNote | Section label | – | – | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Daily Schedule Trigger | scheduleTrigger | Daily trigger at 9 AM | – | Read All Customer Rows | Runs daily at 9:00 AM IST. |
| Read All Customer Rows | googleSheets | Fetch all rows from sheet | Daily Schedule Trigger | Filter Empty Rows | ## 1. Trigger & Data Retrieval — Polls the sheet daily and filters out blank rows. |
| Filter Empty Rows | filter | Remove rows without name/email | Read All Customer Rows | Calculate Date Variables | Skips blank rows. Only rows with a name + email pass through. ## 1. Trigger & Data Retrieval — Polls the sheet daily and filters out blank rows. |
| Calculate Date Variables | set | Compute dynamic date windows | Filter Empty Rows | Is T+3 Email Due?, Is T+5 Email Due?, Is Feedback Email Due?, Is Upsell Email Due? | FIX: t5_window_date is now 10 days (was 9) to catch boundary customers. ## 2. Logic & Scheduling — Calculates dynamic dates and checks email due status. |
| Is T+3 Email Due? | if | Determine if T+3 email should be sent | Calculate Date Variables | Loop T+3 (1 at a time) | ## 2. Logic & Scheduling — Calculates dynamic dates and checks email due status. |
| Loop T+3 (1 at a time) | splitInBatches | Process T+3 items sequentially | Is T+3 Email Due? (true) | Wait (T+3) (output 1), Mark T+3 Email Sent (output 0 loop) | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Wait (T+3) | wait | 20‑second delay before sending | Loop T+3 (1 at a time) | Send T+3 Update Email | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Send T+3 Update Email | zohoZeptomail | Send Day‑3 progress email | Wait (T+3) | Mark T+3 Email Sent | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Mark T+3 Email Sent | googleSheets | Write T3_Email_Sent=YES & original_subject | Send T+3 Update Email | Loop T+3 (1 at a time) | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Is T+5 Email Due? | if | Determine if T+5 email should be sent | Calculate Date Variables | Loop T+5 (1 at a time) | FIX: Requires T3_Email_Sent=YES so T+5 never sends before or at the same time as T+3. ## 2. Logic & Scheduling — Calculates dynamic dates and checks email due status. |
| Loop T+5 (1 at a time) | splitInBatches | Process T+5 items sequentially | Is T+5 Email Due? (true) | Wait (T+5) (output 1), Mark T+5 Email Sent (output 0 loop) | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Wait (T+5) | wait | 20‑second delay before sending | Loop T+5 (1 at a time) | Send T+5 Update Email | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Send T+5 Update Email | zohoZeptomail | Send Day‑5 near‑completion email | Wait (T+5) | Mark T+5 Email Sent | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Mark T+5 Email Sent | googleSheets | Write T5_Email_Sent=YES | Send T+5 Update Email | Loop T+5 (1 at a time) | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Is Feedback Email Due? | if | Determine if feedback email should be sent | Calculate Date Variables | Loop Feedback (1 at a time) | FIX: Added All Done check so completed+upselled customers don't get feedback emails again. ## 2. Logic & Scheduling — Calculates dynamic dates and checks email due status. |
| Loop Feedback (1 at a time) | splitInBatches | Process feedback items sequentially | Is Feedback Email Due? (true) | Wait (Feedback) (output 1), Mark Feedback Email Sent (output 0 loop) | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Wait (Feedback) | wait | 20‑second delay before sending | Loop Feedback (1 at a time) | Send Feedback Request Email | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Send Feedback Request Email | zohoZeptomail | Send feedback request email | Wait (Feedback) | Mark Feedback Email Sent | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Mark Feedback Email Sent | googleSheets | Write Feedback_Email_Sent=YES & Feedback_Email_Date=today | Send Feedback Request Email | Loop Feedback (1 at a time) | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Is Upsell Email Due? | if | Determine if upsell email should be sent | Calculate Date Variables | Loop Upsell (1 at a time) | ## 2. Logic & Scheduling — Calculates dynamic dates and checks email due status. |
| Loop Upsell (1 at a time) | splitInBatches | Process upsell items sequentially | Is Upsell Email Due? (true) | Wait (Upsell) (output 1), Mark Upsell Email Sent (output 0 loop) | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Wait (Upsell) | wait | 20‑second delay before sending | Loop Upsell (1 at a time) | Send Upsell Email1 | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Send Upsell Email1 | zohoZeptomail | Send upsell / cross‑sell email | Wait (Upsell) | Mark Upsell Email Sent | ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |
| Mark Upsell Email Sent | googleSheets | Write Upsell_Email_Sent=YES & Status=All Done | Send Upsell Email1 | Loop Upsell (1 at a time) | Sets Status = All Done after upsell -- stops all future checks for this customer. ## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets. |

---

### 4. Reproducing the Workflow from Scratch

Below is a step‑by‑step guide to recreate the entire workflow manually in a self‑hosted n8n instance.

---

#### Prerequisites

1. **Self‑hosted n8n** (required for community nodes).  
2. Install the community node **n8n‑nodes‑zohozeptomail** via *Settings → Community Nodes → Install* and enter `n8n-nodes-zohozeptomail`.  
3. Create a **Google Sheets OAuth2 credential** with access to the target spreadsheet.  
4. Create a **Zoho ZeptoMail credential** (API key) linked to an active mail agent.  
5. Prepare a Google Sheet with the following exact column headers (case‑sensitive):  
   `Customer Name`, `Email`, `Project Type`, `Date Added`, `Status`, `T3_Email_Sent`, `T5_Email_Sent`, `Feedback_Email_Sent`, `Feedback_Email_Date`, `Upsell_Email_Sent`, `original_subject`, `row_number`.  
   - `row_number` can be a formula `=ROW()` or a static number; it is used as the matching key for updates.  
   - `Date Added` must be in `yyyy-MM-dd` format.  
   - `Status` should contain one of: `In Progress`, `Completed`, or `All Done`.  

---

#### Step‑by‑Step Reconstruction

1. **Create a new workflow** and name it *Send multi-stage customer project emails from Google Sheets with Zoho ZeptoMail*.

2. **Add Sticky Note – Main**  
   - Type: Sticky Note  
   - Content: copy the full text from the Main Sticky node (overview, how it works, setup, customization, important note).  
   - Color: 2 (blue).

3. **Add Sticky Note – Section 1**  
   - Type: Sticky Note  
   - Content: `## 1. Trigger & Data Retrieval — Polls the sheet daily and filters out blank rows.`  
   - Color: 7 (teal).

4. **Add Sticky Note – Section 2**  
   - Type: Sticky Note  
   - Content: `## 2. Logic & Scheduling — Calculates dynamic dates and checks email due status.`  
   - Color: 7.

5. **Add Sticky Note – Section 3**  
   - Type: Sticky Note  
   - Content: `## 3. Communication & Logging — Sends emails via ZeptoMail and updates Google Sheets.`  
   - Color: 7.

6. **Daily Schedule Trigger**  
   - Type: Schedule Trigger (v1.2)  
   - Rule: Interval, `triggerAtHour: 9` (9 AM).  
   - Connect output → **Read All Customer Rows**.

7. **Read All Customer Rows**  
   - Type: Google Sheets (v4.4)  
   - Operation: Read rows  
   - Resource ID mode: ID  
   - Document ID: `YOUR_RESOURCE_ID_HERE` (replace later)  
   - Sheet Name: `YOUR_RESOURCE_ID_HERE` (replace later)  
   - Options: default  
   - Credential: Google Sheets OAuth2  
   - Connect output → **Filter Empty Rows**.

8. **Filter Empty Rows**  
   - Type: Filter (v2)  
   - Combinator: AND  
   - Conditions:  
     - `Email` **not equal** to `""` (left value `={{ $json['Email'] }}`)  
     - `Customer Name` **not equal** to `""` (left value `={{ $json['Customer Name'] }}`)  
   - Type validation: strict, case‑sensitive  
   - Note: “Skips blank rows. Only rows with a name + email pass through.”  
   - Connect true output → **Calculate Date Variables**.

9. **Calculate Date Variables**  
   - Type: Set (v3.4)  
   - Keep other fields: enabled (`includeOtherFields: true`)  
   - Assignments (all type string):  

     | Name | Value |
     |------|-------|
     | today | `={{ $now.toFormat('yyyy-MM-dd') }}` |
     | t3_target_date | `={{ $now.minus({ days: 3 }).toFormat('yyyy-MM-dd') }}` |
     | t3_window_date | `={{ $now.minus({ days: 7 }).toFormat('yyyy-MM-dd') }}` |
     | t5_target_date | `={{ $now.minus({ days: 5 }).toFormat('yyyy-MM-dd') }}` |
     | t5_window_date | `={{ $now.minus({ days: 10 }).toFormat('yyyy-MM-dd') }}` |
     | upsell_target_date | `={{ $now.minus({ days: 3 }).toFormat('yyyy-MM-dd') }}` |
     | upsell_window_date | `={{ $now.minus({ days: 7 }).toFormat('yyyy-MM-dd') }}` |

   - Note: “FIX: t5_window_date is now 10 days (was 9) to catch boundary customers.”  
   - Connect output to the four IF nodes: **Is T+3 Email Due?**, **Is T+5 Email Due?**, **Is Feedback Email Due?**, **Is Upsell Email Due?**.

10. **Is T+3 Email Due?**  
    - Type: IF (v2.1)  
    - Loose type validation: enabled  
    - Combinator: AND  
    - Conditions:  

      1. `Date Added` **after** `t3_window_date`  
      2. `Date Added` **before or equals** `t3_target_date`  
      3. `T3_Email_Sent` **empty**  
      4. `Status` **not equals** `Completed`  
      5. `Status` **not equals** `All Done`  

    - Connect **true** output → **Loop T+3 (1 at a time)**.

11. **Loop T+3 (1 at a time)**  
    - Type: Split In Batches (v3)  
    - Connect output 1 → **Wait (T+3)**.  
    - Output 0 (loop completion) will later be wired from **Mark T+3 Email Sent** back into this node.

12. **Wait (T+3)**  
    - Type: Wait (v1.1)  
    - Amount: 20 (seconds)  
    - Connect output → **Send T+3 Update Email**.

13. **Send T+3 Update Email**  
    - Type: Zoho ZeptoMail (community node)  
    - Parameters:  

      - `subject`: `={{ 'Update on Your ' + $json['Project Type'] + ' Project - We\'re Making Progress!' }}`  
      - `htmlbody`: paste the Day‑3 HTML template (see JSON). It references `$json['Customer Name']` and `$json['Project Type']`.  
      - `mailagent`: `555046c28339d27e` (replace with your mail agent ID)  
      - `toaddress`: `={{ $json['Email'] }}`  
      - `fromaddress`: `{ name: "Think Sage", address: "YOUR_EMAIL_HERE" }` (replace)  

    - Credential: Zoho ZeptoMail  
    - Connect output → **Mark T+3 Email Sent**.

14. **Mark T+3 Email Sent**  
    - Type: Google Sheets (v4.4)  
    - Operation: Update  
    - Document ID / Sheet Name: same placeholders as earlier  
    - Matching column: `row_number`  
    - Column mappings (preserve all original columns, override only):  

      - `T3_Email_Sent` = `YES`  
      - `original_subject` = `={{ 'Update on Your ' + $('Loop T+3 (1 at a time)').item.json['Project Type'] + ' Project - We\'re Making Progress!' }}`  

    - Credential: Google Sheets OAuth2  
    - Connect output → **Loop T+3 (1 at a time)** (input for next batch).

15. **Is T+5 Email Due?**  
    - Type: IF (v2.1)  
    - Strict type validation  
    - Combinator: AND  
    - Conditions:  

      1. `Date Added` **after** `t5_window_date`  
      2. `Date Added` **before or equals** `t5_target_date`  
      3. `T5_Email_Sent` **empty**  
      4. `Status` **not equals** `Completed`  
      5. `Status` **not equals** `All Done`  
      6. `T3_Email_Sent` **equals** `YES`  

    - Note: “FIX: Requires T3_Email_Sent=YES so T+5 never sends before or at the same time as T+3.”  
    - Connect **true** output → **Loop T+5 (1 at a time)**.

16. **Loop T+5 (1 at a time)** – same pattern as T+3 loop.  
    - Connect output 1 → **Wait (T+5)**.

17. **Wait (T+5)** – 20 seconds → **Send T+5 Update Email**.

18. **Send T+5 Update Email**  
    - Type: Zoho ZeptoMail  
    - `subject`: `={{ $json['original_subject'] }}`  
    - `htmlbody`: Day‑5 HTML template (see JSON)  
    - `mailagent`, `fromaddress`, `toaddress` – same pattern, replace `YOUR_EMAIL_HERE`  
    - Connect → **Mark T+5 Email Sent**.

19. **Mark T+5 Email Sent**  
    - Type: Google Sheets update  
    - Matching column: `row_number`  
    - Override: `T5_Email_Sent` = `YES` (all other columns from loop item)  
    - Connect → **Loop T+5 (1 at a time)**.

20. **Is Feedback Email Due?**  
    - Type: IF (v2.1) – strict  
    - Conditions (AND):  

      1. `Status` **not equals** `In Progress`  
      2. `Status` **not equals** `All Done`  
      3. `Feedback_Email_Sent` **empty**  
      4. `T3_Email_Sent` **equals** `YES`  
      5. `T5_Email_Sent` **equals** `YES`  

    - Note: “FIX: Added All Done check so completed+upselled customers don't get feedback emails again.”  
    - Connect **true** → **Loop Feedback (1 at a time)**.

21. **Loop Feedback (1 at a time)** → **Wait (Feedback)** → **Send Feedback Request Email** → **Mark Feedback Email Sent** → back to loop.  
    - **Send Feedback Request Email**: subject uses `original_subject`; body contains a CTA button linking to `YOUR_FEEDBACK_FORM_LINK` (replace).  
    - **Mark Feedback Email Sent**: set `Feedback_Email_Sent = YES` and `Feedback_Email_Date = {{ $now.toFormat('yyyy-MM-dd') }}`.

22. **Is Upsell Email Due?**  
    - Type: IF (v2.1) – strict  
    - Conditions (AND):  

      1. `Status` **not equals** `In Progress`  
      2. `Status` **not equals** `All Done`  
      3. `Feedback_Email_Sent` **equals** `YES`  
      4. `Feedback_Email_Date` **after** `upsell_window_date`  
      5. `Feedback_Email_Date` **before or equals** `upsell_target_date`  
      6. `Upsell_Email_Sent` **empty**  

    - Connect **true** → **Loop Upsell (1 at a time)**.

23. **Loop Upsell (1 at a time)** → **Wait (Upsell)** → **Send Upsell Email1** → **Mark Upsell Email Sent** → back to loop.  
    - **Send Upsell Email1**: subject reuses `original_subject`; body includes an upsell table and CTA linking to `YOUR_BOOKING_LINK` (replace). Discount code `RETURN15` is hardcoded.  
    - **Mark Upsell Email Sent**: set `Upsell_Email_Sent = YES` and **override `Status` to `All Done`**. Note on node: “Sets Status = All Done after upsell -- stops all future checks for this customer.”

24. **Replace Placeholders**  
    - In every Google Sheets node, replace `YOUR_RESOURCE_ID_HERE` with your actual spreadsheet document ID and sheet ID.  
    - In every ZeptoMail node, replace `YOUR_EMAIL_HERE` with your verified sender address.  
    - In the Feedback email body, replace `YOUR_FEEDBACK_FORM_LINK` with your real form URL.  
    - In the Upsell email body, replace `YOUR_BOOKING_LINK` with your real booking URL.  
    - If your ZeptoMail mail agent ID differs from `555046c28339d27e`, update it in all four email nodes.

25. **Activate the workflow**. After activation, it will execute daily at 9 AM (server timezone). Manually execute once to verify credentials and placeholder replacements.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow requires the community node `n8n-nodes-zohozeptomail`. Install it before importing. | n8n Community Nodes settings |
| All date calculations use `$now`, which respects the n8n server timezone. Confirm the timezone matches your business operating hours. | n8n environment variable `GENERIC_TIMEZONE` |
| The `row_number` column must exist in the sheet and contain a unique numeric identifier for each row; it is used as the matching key when updating rows. | Google Sheets best practices |
| ZeptoMail mail agent ID (`555046c28339d27e`) is specific to the original author’s account. Replace it with your own mail agent ID from the ZeptoMail dashboard. | Zoho ZeptoMail configuration |
| The Wait nodes pause for 20 seconds between emails. Adjust this value if your ZeptoMail plan has stricter rate limits. | ZeptoMail rate limits documentation |
| The `original_subject` column is populated when the T+3 email is sent and then reused by all subsequent emails for that customer, ensuring a consistent email thread. | Email threading best practice |
| Status values recognized by the workflow are `In Progress`, `Completed`, and `All Done`. Any other value will be treated as neither, potentially causing unexpected branch behavior. | Sheet data validation |
| The workflow’s logic ensures emails are sent in strict order: T+3 → T+5 → Feedback → Upsell. Skipping a stage (e.g., manually marking T5_Email_Sent=YES without T3) may produce inconsistent threading or subject lines. | Process integrity note |
| For businesses outside the resume/LinkedIn niche, customize the HTML templates in the four ZeptoMail nodes to match your service offering and branding. | Template customization |
| The discount code `RETURN15` in the upsell email is hardcoded. If you change it, remember to update both the email body and any linked promotion system. | Promo code management |
| The feedback email contains a placeholder link (`YOUR_FEEDBACK_FORM_LINK`). Use a form tool (Google Forms, Typeform, etc.) and paste the direct link. | Form integration |
| The upsell email contains a placeholder booking link (`YOUR_BOOKING_LINK`). Replace with your scheduling page (Calendly, Acuity, etc.). | Scheduling integration |
| The workflow description and setup instructions are also present in the Main Sticky note for quick in‑canvas reference. | In‑workflow documentation |
| Project credit: Workflow designed for service businesses that manage client orders in Google Sheets and want structured follow‑up without a CRM. | Original workflow description |