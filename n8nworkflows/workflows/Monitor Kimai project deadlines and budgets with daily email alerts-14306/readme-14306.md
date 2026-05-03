Monitor Kimai project deadlines and budgets with daily email alerts

https://n8nworkflows.xyz/workflows/monitor-kimai-project-deadlines-and-budgets-with-daily-email-alerts-14306


# Monitor Kimai project deadlines and budgets with daily email alerts

# 1. Workflow Overview

This workflow monitors **billable Kimai projects** on a **weekday schedule at 9:00 AM** and sends an **HTML email alert** when at least one project meets one of these criteria:

- its **project end date** is within the next **10 days**
- its **time budget usage** is at or above **80%**

If no project matches the alert conditions, the workflow ends without sending any email.

## 1.1 Scheduled Start

The workflow starts from a schedule trigger that runs every Monday to Friday at 09:00.

## 1.2 Project Retrieval from Kimai

It fetches all visible Kimai projects, filters them to keep only **billable** ones, then retrieves:
- detailed project information for each billable project
- timesheet entries associated with each billable project

## 1.3 Budget Aggregation and Data Merge

The workflow aggregates timesheet durations per project to estimate used budget in hours/seconds, then merges that usage data with the detailed project records.

## 1.4 Deadline and Budget Evaluation

The merged project dataset is analyzed to determine:
- deadline urgency
- number of days remaining until project end
- budget usage percentage
- whether the project should be included in the alert report

Projects that do not meet alert criteria are discarded.

## 1.5 Conditional Email Generation and Sending

If at least one project needs attention, the workflow generates a styled HTML email report and sends it via SMTP. If no projects need reporting, no message is sent.

---

# 2. Block-by-Block Analysis

## Block 1 — Scheduled Start

### Overview
This block provides the workflow entry point. It ensures the process runs automatically on weekdays at a fixed time.

### Nodes Involved
- Every Day at 9:00

### Node Details

#### Every Day at 9:00
- **Type / role:** `n8n-nodes-base.scheduleTrigger` — workflow trigger
- **Technical role:** Starts the workflow automatically on a cron-based schedule.
- **Configuration choices:**
  - Uses a cron expression: `0 9 * * 1-5`
  - Runs at 09:00 on Monday through Friday
- **Key expressions or variables used:** None
- **Input connections:** None; this is an entry point
- **Output connections:** Sends output to `GET Projects`
- **Version-specific requirements:** Type version `1.2`
- **Edge cases / failures:**
  - Workflow must be active for the trigger to run
  - Server timezone affects actual execution time unless controlled at instance/workflow level
- **Sub-workflow reference:** None

---

## Block 2 — Retrieve and Prepare Kimai Data

### Overview
This block fetches project data from Kimai and narrows the dataset to billable projects only. It then branches into project-detail retrieval and timesheet retrieval so that deadline and budget information can later be combined.

### Nodes Involved
- GET Projects
- Get only Bilable
- GET Projects Details
- GET Timesheet Records
- Calculate Budget Uses
- Combine Data

### Node Details

#### GET Projects
- **Type / role:** `n8n-nodes-base.httpRequest` — API request to Kimai
- **Technical role:** Fetches visible projects from Kimai.
- **Configuration choices:**
  - Method defaults to GET
  - URL: `https://kimai/api/projects`
  - Query parameter `visible=1`
  - Authentication uses **generic credential type** with **HTTP Bearer Auth**
- **Key expressions or variables used:** None
- **Input connections:** `Every Day at 9:00`
- **Output connections:** `Get only Bilable`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases / failures:**
  - Bearer token missing/invalid
  - Incorrect base URL
  - SSL/TLS problems if Kimai uses custom certificates
  - Kimai API pagination behavior could affect completeness if the endpoint paginates by default
- **Sub-workflow reference:** None

#### Get only Bilable
- **Type / role:** `n8n-nodes-base.code` — filter logic
- **Technical role:** Keeps only projects where `billable === true`.
- **Configuration choices:**
  - JavaScript loops through `$input.all()`
  - Returns only items whose `item.json.billable` is exactly `true`
- **Key expressions or variables used:**
  - `$input.all()`
  - `item.json.billable`
- **Input connections:** `GET Projects`
- **Output connections:** 
  - `GET Projects Details`
  - `GET Timesheet Records`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - Node name contains a typo: “Bilable” instead of “Billable”; this is cosmetic only
  - If Kimai returns billable values in another format, strict boolean comparison may exclude valid projects
  - If no billable projects exist, downstream nodes receive no items
- **Sub-workflow reference:** None

#### GET Projects Details
- **Type / role:** `n8n-nodes-base.httpRequest` — per-project detail fetch
- **Technical role:** Retrieves the full details for each billable project.
- **Configuration choices:**
  - URL expression: `https://kimai/api/projects/{{$json.id}}`
  - Authentication via HTTP Bearer Auth
- **Key expressions or variables used:**
  - `{{$json.id}}`
- **Input connections:** `Get only Bilable`
- **Output connections:** `Combine Data` as input 1
- **Version-specific requirements:** Type version `4.2`
- **Edge cases / failures:**
  - Missing `id` in input data causes malformed URL
  - Project may have been deleted between list and detail calls
  - Authentication / network / API format errors
- **Sub-workflow reference:** None

#### GET Timesheet Records
- **Type / role:** `n8n-nodes-base.httpRequest` — timesheet fetch
- **Technical role:** Retrieves timesheet records for each billable project.
- **Configuration choices:**
  - URL: `https://kimai/api/timesheets`
  - Query parameters:
    - `user=all`
    - `project={{ $json.id }}`
    - `size=1+1234567890`
  - Uses HTTP Bearer Auth
- **Key expressions or variables used:**
  - `={{ $json.id }}`
- **Input connections:** `Get only Bilable`
- **Output connections:** `Calculate Budget Uses`
- **Version-specific requirements:** Type version `4.2`
- **Edge cases / failures:**
  - The `size` value is unusual: `1+1234567890`
    - Depending on Kimai/API parsing, this may not behave as intended
    - It appears designed to request a very large page size
  - If pagination is enforced server-side, returned data may still be incomplete
  - If project IDs are missing, query becomes invalid
  - Large projects may produce slow or heavy API responses
- **Sub-workflow reference:** None

#### Calculate Budget Uses
- **Type / role:** `n8n-nodes-base.code` — aggregation logic
- **Technical role:** Sums timesheet `duration` values by project and converts totals into seconds and hours.
- **Configuration choices:**
  - Iterates through all timesheet items
  - Determines `projectId` from either `item.json.project.id` or `item.json.project`
  - Adds `duration`
  - Returns one item per project with:
    - `id`
    - `total_seconds`
    - `total_ore`
- **Key expressions or variables used:**
  - `$input.all()`
  - `item.json.project?.id`
  - `item.json.duration`
- **Input connections:** `GET Timesheet Records`
- **Output connections:** `Combine Data` as input 2
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - If timesheet entries are absent, the output may be empty for some projects
  - Merge behavior later may leave projects without usage totals
  - `total_ore` is calculated but not used later; harmless but redundant
  - Non-numeric durations are coerced to `0`
- **Sub-workflow reference:** None

#### Combine Data
- **Type / role:** `n8n-nodes-base.merge` — dataset merge
- **Technical role:** Enriches project detail items with aggregated budget usage data using the project `id`.
- **Configuration choices:**
  - Mode: `combine`
  - Join mode: `enrichInput1`
  - Match field: `id`
  - Input 1 = project details
  - Input 2 = budget usage data
- **Key expressions or variables used:** Matching on field `id`
- **Input connections:**
  - `GET Projects Details`
  - `Calculate Budget Uses`
- **Output connections:** `Calculate expiration`
- **Version-specific requirements:** Type version `3.2`
- **Edge cases / failures:**
  - If no matching budget item exists, project detail still passes through because of `enrichInput1`
  - Type mismatches on `id` can break joins if one side is string and the other numeric; here the code normalizes to number, which helps
  - If upstream API calls complete with inconsistent counts, output may omit usage enrichment for some projects
- **Sub-workflow reference:** None

---

## Block 3 — Evaluate Deadlines and Budget Risk

### Overview
This block computes deadline urgency, budget utilization, and report inclusion rules. It also condenses all qualifying projects into a single summary item for downstream email generation.

### Nodes Involved
- Calculate expiration
- Need Email?

### Node Details

#### Calculate expiration
- **Type / role:** `n8n-nodes-base.code` — business-rule evaluation
- **Technical role:** Determines which projects should be reported based on deadline proximity and budget consumption.
- **Configuration choices:**
  - Sets `daysThreshold = 10`
  - Normalizes today to midnight to avoid partial-day drift
  - Computes:
    - formatted deadline
    - ISO deadline
    - days remaining
    - urgency level (`expired`, `high`, `medium`, `low`, `none`)
    - project status (`expired`, `expiring`, `not_present`)
  - Computes budget info from:
    - `timeBudget`
    - `total_seconds`
  - Budget report threshold is implemented in `getBudgetInfo()` as `percentage >= 80`
  - Includes a project in the final report if:
    - `projectDays` is between 0 and `daysThreshold`, or
    - budget usage should be reported
  - Returns a single item:
    - `daysThreshold`
    - `count`
    - `projects` array
- **Key expressions or variables used:**
  - `$input.all()`
  - `item.json.end`
  - `item.json.timeBudget`
  - `item.json.total_seconds`
  - `item.json.parentTitle`
- **Input connections:** `Combine Data`
- **Output connections:** `Need Email?`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - **Expired deadlines are not included** unless budget threshold is also met:
    - `projectShouldReport` requires `projectDays >= 0`
    - this means projects already past due are excluded from date-based alerting
    - however the email renderer includes display logic for expired items, so this may be an unintended inconsistency
  - Projects with no deadline are also excluded unless budget threshold is met
  - Invalid date strings become “not present”-like behavior
  - Division by zero is avoided because budgets `<= 0` return `null`
  - Negative remaining hours are possible when over budget; this is valid but should be interpreted carefully
- **Sub-workflow reference:** None

#### Need Email?
- **Type / role:** `n8n-nodes-base.if` — conditional branch
- **Technical role:** Checks whether at least one project needs reporting.
- **Configuration choices:**
  - Condition: `$json.count > 0`
  - Only the true branch is connected
- **Key expressions or variables used:**
  - `={{ $json.count }}`
- **Input connections:** `Calculate expiration`
- **Output connections:** `Build Email HTML - Report` on true branch
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - If `count` is missing or non-numeric, strict validation may fail or evaluate unexpectedly
  - False branch is intentionally unused, so the workflow silently stops when no alerts are needed
- **Sub-workflow reference:** None

---

## Block 4 — Build and Send the Alert Email

### Overview
This block transforms the project summary into a polished HTML email and sends it via SMTP. It only runs if at least one project requires attention.

### Nodes Involved
- Build Email HTML - Report
- Send an Email

### Node Details

#### Build Email HTML - Report
- **Type / role:** `n8n-nodes-base.code` — HTML report generator
- **Technical role:** Produces the email subject and HTML body from the aggregated project array.
- **Configuration choices:**
  - Reads:
    - `projects`
    - `count`
    - `daysThreshold`
  - Uses `$now.setLocale('en').toFormat('dd LLL yyyy')` for the footer date
  - Builds visual status cards for deadline urgency
  - Builds a budget card with a progress bar
  - Escapes HTML in project/customer labels with a helper
  - Returns:
    - `html`
    - `subject`
- **Key expressions or variables used:**
  - `$json.projects`
  - `$json.count`
  - `$json.daysThreshold`
  - `$now.setLocale('en').toFormat(...)`
- **Input connections:** `Need Email?`
- **Output connections:** `Send an Email`
- **Version-specific requirements:** Type version `2`
- **Edge cases / failures:**
  - Relies on Luxon-style `$now` formatting support in n8n expressions/runtime
  - If `projects` is empty despite passing the IF, the email will render with no rows
  - CTA link is hardcoded to `https://kimai.com`, not the local Kimai instance
  - `daysThreshold` is read but not displayed in the final content
- **Sub-workflow reference:** None

#### Send an Email
- **Type / role:** `n8n-nodes-base.emailSend` — SMTP sender
- **Technical role:** Sends the generated HTML report email.
- **Configuration choices:**
  - Subject from `{{$json.subject}}`
  - HTML from `{{$json.html}}`
  - To: `info@example.com`
  - From: `info@example.com`
  - SMTP credential: `SMTP account`
  - Options:
    - `appendAttribution: false`
    - `allowUnauthorizedCerts: true`
- **Key expressions or variables used:**
  - `={{ $json.subject }}`
  - `={{ $json.html }}`
- **Input connections:** `Build Email HTML - Report`
- **Output connections:** None
- **Version-specific requirements:** Type version `2.1`
- **Edge cases / failures:**
  - SMTP auth or connection failure
  - Sender address may need to match authenticated SMTP identity
  - `allowUnauthorizedCerts: true` weakens TLS validation; useful for internal SMTP but risky in production
  - Invalid HTML or oversized email bodies may be rejected by some mail servers
- **Sub-workflow reference:** None

---

## Block 5 — Documentation / Sticky Notes

### Overview
These nodes do not participate in execution. They document workflow sections and highlight customization points.

### Nodes Involved
- Sticky Note
- Sticky Note1
- Sticky Note2
- Sticky Note3
- Sticky Note4
- Sticky Note5

### Node Details

#### Sticky Note
- **Type / role:** `n8n-nodes-base.stickyNote` — canvas documentation
- **Technical role:** Notes where to customize the day-range threshold.
- **Configuration choices:** Contains text: `To Customize the Range of Day to check customize here.`
- **Input connections:** None
- **Output connections:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note1
- **Type / role:** `n8n-nodes-base.stickyNote`
- **Technical role:** Labels the schedule section.
- **Configuration choices:** `## 1. Scheduled Start`
- **Input / output:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note2
- **Type / role:** `n8n-nodes-base.stickyNote`
- **Technical role:** Labels the Kimai retrieval section.
- **Configuration choices:** `## 2.  Get Information from Kimai`
- **Input / output:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note3
- **Type / role:** `n8n-nodes-base.stickyNote`
- **Technical role:** Labels the expiration-check section.
- **Configuration choices:** `## 3. Check Expiration`
- **Input / output:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note4
- **Type / role:** `n8n-nodes-base.stickyNote`
- **Technical role:** Labels the email build/send section.
- **Configuration choices:** `## 4. Build & Send Email`
- **Input / output:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

#### Sticky Note5
- **Type / role:** `n8n-nodes-base.stickyNote`
- **Technical role:** Provides a full workflow description and customization map.
- **Configuration choices:** Contains:
  - title and purpose
  - schedule summary
  - alert criteria
  - customization table
- **Input / output:** None
- **Version-specific requirements:** Type version `1`
- **Edge cases / failures:** None
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Every Day at 9:00 | Schedule Trigger | Starts the workflow every weekday at 9 AM |  | GET Projects | ## 1. Scheduled Start<br># 📅 Kimai — Deadline & Budget Monitor<br><br>## Monitors billable Kimai projects daily and sends an HTML alert email when a deadline is within 10 days or the hour budget exceeds 80%.<br><br>Runs every weekday at **9 AM**.<br>Fetches all billable projects and checks:<br>- End date within **10 days**<br>- Budget consumption **≥ 80%**<br><br>No alerts needed? No email sent.<br><br>## ⚙️ Customize<br><br>| What | Where |<br>|---|---|<br>| Days threshold | `Calculate expiration` → line 1 |<br>| Budget % alert | `Calculate expiration` → `getBudgetInfo()` |<br>| Kimai URL | All 3 HTTP Request nodes |<br>| Sender / Recipient | `Send an Email` node | |
| GET Projects | HTTP Request | Fetches visible projects from Kimai | Every Day at 9:00 | Get only Bilable | ## 2.  Get Information from Kimai |
| Get only Bilable | Code | Filters projects to retain only billable ones | GET Projects | GET Projects Details, GET Timesheet Records | ## 2.  Get Information from Kimai |
| GET Projects Details | HTTP Request | Retrieves detailed data for each billable project | Get only Bilable | Combine Data | ## 2.  Get Information from Kimai |
| GET Timesheet Records | HTTP Request | Retrieves timesheet records for each billable project | Get only Bilable | Calculate Budget Uses | ## 2.  Get Information from Kimai |
| Calculate Budget Uses | Code | Aggregates timesheet durations per project | GET Timesheet Records | Combine Data | ## 2.  Get Information from Kimai |
| Combine Data | Merge | Joins project details with budget usage totals | GET Projects Details, Calculate Budget Uses | Calculate expiration | ## 2.  Get Information from Kimai |
| Calculate expiration | Code | Evaluates deadline proximity and budget alert conditions | Combine Data | Need Email? | ## 3. Check Expiration<br>To Customize the Range of Day to check customize here. |
| Need Email? | If | Sends flow forward only if at least one project needs reporting | Calculate expiration | Build Email HTML - Report | ## 4. Build & Send Email |
| Build Email HTML - Report | Code | Generates HTML email body and subject | Need Email? | Send an Email | ## 4. Build & Send Email |
| Send an Email | Email Send | Sends the final HTML alert email via SMTP | Build Email HTML - Report |  | ## 4. Build & Send Email |
| Sticky Note | Sticky Note | Canvas annotation for day-range customization |  |  |  |
| Sticky Note1 | Sticky Note | Canvas annotation for schedule section |  |  |  |
| Sticky Note2 | Sticky Note | Canvas annotation for Kimai information section |  |  |  |
| Sticky Note3 | Sticky Note | Canvas annotation for expiration section |  |  |  |
| Sticky Note4 | Sticky Note | Canvas annotation for email section |  |  |  |
| Sticky Note5 | Sticky Note | Canvas annotation with overall workflow description and customization hints |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** in n8n and name it something like:  
   **Monitor Kimai project deadlines and budgets with daily email alerts**.

2. **Add a Schedule Trigger node**
   - Type: **Schedule Trigger**
   - Name: `Every Day at 9:00`
   - Configure with cron mode:
     - Expression: `0 9 * * 1-5`
   - This makes it run Monday to Friday at 09:00.

3. **Add an HTTP Request node to list Kimai projects**
   - Type: **HTTP Request**
   - Name: `GET Projects`
   - Method: `GET`
   - URL: `https://kimai/api/projects`
   - Enable query parameters:
     - `visible` = `1`
   - Authentication:
     - Choose **Generic Credential Type**
     - Auth type: **HTTP Bearer Auth**
     - Create/select credentials for your Kimai API token
   - Connect `Every Day at 9:00` → `GET Projects`

4. **Add a Code node to keep only billable projects**
   - Type: **Code**
   - Name: `Get only Bilable`
   - Paste this logic conceptually:
     - loop through all incoming items
     - keep only items where `billable === true`
     - return the filtered array
   - Connect `GET Projects` → `Get only Bilable`

5. **Add an HTTP Request node to fetch per-project details**
   - Type: **HTTP Request**
   - Name: `GET Projects Details`
   - Method: `GET`
   - URL expression:
     - `https://kimai/api/projects/{{$json.id}}`
   - Authentication:
     - same Kimai HTTP Bearer credential
   - Connect `Get only Bilable` → `GET Projects Details`

6. **Add another HTTP Request node to fetch project timesheets**
   - Type: **HTTP Request**
   - Name: `GET Timesheet Records`
   - Method: `GET`
   - URL: `https://kimai/api/timesheets`
   - Authentication:
     - same Kimai HTTP Bearer credential
   - Enable query parameters:
     - `user` = `all`
     - `project` = `={{ $json.id }}`
     - `size` = `1+1234567890`
   - Connect `Get only Bilable` → `GET Timesheet Records`

7. **Add a Code node to aggregate timesheet duration by project**
   - Type: **Code**
   - Name: `Calculate Budget Uses`
   - Logic to implement:
     - read all timesheet items
     - extract project ID from `item.json.project.id` or fallback to `item.json.project`
     - sum `duration` values per project
     - return one item per project with:
       - `id`
       - `total_seconds`
       - `total_ore`
   - Connect `GET Timesheet Records` → `Calculate Budget Uses`

8. **Add a Merge node to enrich project details with usage totals**
   - Type: **Merge**
   - Name: `Combine Data`
   - Mode: **Combine**
   - Join mode: **Enrich Input 1**
   - Field to match: `id`
   - Connect:
     - `GET Projects Details` → `Combine Data` input 1
     - `Calculate Budget Uses` → `Combine Data` input 2

9. **Add a Code node for business-rule evaluation**
   - Type: **Code**
   - Name: `Calculate expiration`
   - Configure logic to:
     - define `daysThreshold = 10`
     - compute current date at midnight
     - format project end date from ISO to `dd/mm/yyyy`
     - calculate days remaining
     - classify urgency:
       - expired
       - high
       - medium
       - low
       - none
     - compute budget metrics from `timeBudget` and `total_seconds`
     - mark a project for reporting when:
       - end date is within threshold and not already past, or
       - budget usage is at least 80%
     - return a single item containing:
       - `daysThreshold`
       - `count`
       - `projects` array
   - Connect `Combine Data` → `Calculate expiration`

10. **Add an IF node to decide whether an email is needed**
    - Type: **If**
    - Name: `Need Email?`
    - Condition:
      - left value: `={{ $json.count }}`
      - operator: `greater than`
      - right value: `0`
    - Connect `Calculate expiration` → `Need Email?`

11. **Add a Code node to generate HTML**
    - Type: **Code**
    - Name: `Build Email HTML - Report`
    - Logic to implement:
      - read `projects`, `count`, and `daysThreshold`
      - generate a human-friendly HTML email
      - create one visual block per project
      - include:
        - customer name
        - project name
        - deadline status
        - budget usage progress bar
      - create a subject such as:
        - `[Timesheet] - Projects Deadline Report`
      - return:
        - `html`
        - `subject`
    - Connect the **true** output of `Need Email?` → `Build Email HTML - Report`

12. **Add an Email Send node**
    - Type: **Send Email**
    - Name: `Send an Email`
    - Configure:
      - To: `info@example.com`
      - From: `info@example.com`
      - Subject: `={{ $json.subject }}`
      - HTML: `={{ $json.html }}`
    - Options:
      - `appendAttribution = false`
      - `allowUnauthorizedCerts = true`
    - Credentials:
      - create/select an **SMTP** credential
    - Connect `Build Email HTML - Report` → `Send an Email`

13. **Add optional sticky notes for documentation**
    - Add canvas notes for:
      - Scheduled Start
      - Get Information from Kimai
      - Check Expiration
      - Build & Send Email
      - overall workflow summary and customization points

14. **Activate the workflow**
    - Ensure:
      - Kimai bearer token works
      - SMTP credential works
      - URLs point to your real Kimai instance
      - recipient/sender addresses are valid

15. **Test manually before production**
    - Run the workflow manually
    - Confirm that:
      - projects are retrieved
      - only billable projects continue
      - timesheet totals merge correctly
      - count becomes greater than zero when expected
      - email HTML renders correctly
      - no email is sent when `count = 0`

## Credential Setup Details

### Kimai API Credential
- Use **HTTP Bearer Auth**
- Supply the Kimai API token/service account token
- Reuse this credential in:
  - `GET Projects`
  - `GET Projects Details`
  - `GET Timesheet Records`

### SMTP Credential
- Use your mail server host, port, username, password, and security settings
- Reuse this credential in `Send an Email`

## Important Rebuild Constraints

- Replace all `https://kimai/...` URLs with your actual Kimai base URL if needed
- The current workflow assumes Kimai API responses include:
  - project `id`
  - `billable`
  - `end`
  - `timeBudget`
  - `parentTitle`
- The workflow has **one entry point only**: the schedule trigger
- There are **no sub-workflows** and no “Execute Workflow” nodes

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Kimai — Deadline & Budget Monitor | Workflow branding/title |
| Monitors billable Kimai projects daily and sends an HTML alert email when a deadline is within 10 days or the hour budget exceeds 80%. | Functional summary |
| Runs every weekday at 9 AM. | Schedule behavior |
| Fetches all billable projects and checks: End date within 10 days; Budget consumption ≥ 80%. | Alert criteria |
| No alerts needed? No email sent. | Conditional behavior |
| Days threshold can be customized in `Calculate expiration` on line 1. | Customization note |
| Budget percentage alert can be customized in `Calculate expiration` inside `getBudgetInfo()`. | Customization note |
| Kimai URL must be adjusted in all 3 HTTP Request nodes. | Deployment note |
| Sender / Recipient must be adjusted in `Send an Email`. | Deployment note |
| Kimai website link used in the email CTA. | https://kimai.com |

## Additional implementation observations
- The workflow name and visual design suggest deadline monitoring, but it also performs **budget exhaustion monitoring**.
- The code supports displaying expired projects, but the current inclusion rule excludes expired deadlines unless the budget threshold is also exceeded.
- `allowUnauthorizedCerts: true` is convenient for internal infrastructure, but should be reviewed for security before production use.
- The `size=1+1234567890` parameter in the timesheet request should be validated against your Kimai API behavior.