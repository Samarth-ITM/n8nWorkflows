Send weekly TMetric time reports to employees on Slack

https://n8nworkflows.xyz/workflows/send-weekly-tmetric-time-reports-to-employees-on-slack-14511


# Send weekly TMetric time reports to employees on Slack

# 1. Workflow Overview

This workflow sends weekly Slack reports to employees based on their TMetric activity. It compares each employee’s worked time against their expected work schedule, accounts for approved time off, summarizes project activity, and then delivers a personalized Slack message.

It also contains a separate setup path that must be run manually before scheduled use. The setup creates or reuses an n8n Data Table that maps TMetric user IDs to Slack user IDs, then asks Slack users to identify their TMetric account through a form.

## 1.1 Entry Points and Mode Selection

The workflow has two entry points:

- **Run Once a Week**: scheduled execution for normal reporting
- **Setup**: manual execution for creating/updating the TMetric-to-Slack user mapping

Both entry points pass through **Globals**, then **Is running Setup?**, which routes execution based on whether the input resembles manual setup data or schedule trigger data.

## 1.2 Scheduled Reporting Data Collection

In the reporting branch, the workflow retrieves three TMetric datasets:

- users
- work schedules
- time off requests

It then reshapes and joins these datasets into a per-user repository object.

## 1.3 Per-User Processing and Metrics Calculation

Each prepared user record is processed individually. For each user, the workflow:

- loads the Slack mapping table
- fetches that user’s TMetric time entries
- removes running tasks
- aggregates time entries
- merges everything together
- computes schedule compliance and project summaries

A **Wait** node introduces a pause between cycles to reduce the risk of rate limiting.

## 1.4 Slack Delivery

The workflow merges the computed per-user report with the Slack ID mapping from the Data Table, then sends a personalized Slack message using Block Kit JSON.

## 1.5 Manual Setup for User Mapping

The setup branch ensures the required Data Table exists, gets TMetric users and Slack users, filters Slack accounts, and sends each Slack user a form asking them to select their TMetric account. Responses are stored in the Data Table using an upsert operation.

---

# 2. Block-by-Block Analysis

## 2.1 Entry Points, Global Variables, and Routing

### Overview
This block initializes shared variables and decides whether the workflow is executing in scheduled-report mode or setup mode. The routing depends on the trigger payload shape rather than a dedicated flag.

### Nodes Involved
- Run Once a Week
- Setup
- Globals
- Is running Setup?

### Node Details

#### Run Once a Week
- **Type and role:** `scheduleTrigger`; starts the reporting branch automatically once per week.
- **Configuration:** Runs every week on day `5` at hour `16`. In n8n schedule semantics, the exact weekday interpretation depends on instance locale/version, so verify in UI.
- **Expressions/variables:** none.
- **Input/Output:** entry node; outputs to **Globals**.
- **Version notes:** typeVersion `1.1`.
- **Edge cases:** timezone mismatch, unexpected weekday interpretation, disabled workflow, instance clock issues.

#### Setup
- **Type and role:** `manualTrigger`; starts the mapping/setup branch on demand.
- **Configuration:** no parameters.
- **Input/Output:** entry node; outputs to **Globals**.
- **Version notes:** typeVersion `1`.
- **Edge cases:** manual-only; cannot run automatically.

#### Globals
- **Type and role:** `set`; stores reusable constants and computed date boundaries.
- **Configuration choices:**
  - `tmAccountId`: must be filled manually
  - `Allow hours missing percentage`: threshold for warning users, default `10`
  - `tmetricToSlackUserDataTableName`: default `"Tmetric to Slack user map"`
  - `apiBaseUrl`: `https://app.tmetric.com/api/v3`
  - `fullEndDate`: current date
  - `beginningOfTheMonth`: first day of current month
  - `endOfTheMonth`: last day of current month
- **Key expressions:**
  - `{{$now.format('yyyy-LL-dd')}}`
  - Luxon-style date expressions for first/last day of month
- **Input/Output:** receives from both triggers; outputs to **Is running Setup?**
- **Version notes:** typeVersion `3.4`.
- **Edge cases:** empty `tmAccountId` breaks all TMetric API calls; date functions depend on n8n expression runtime; month rollover logic should be validated around December/January.

#### Is running Setup?
- **Type and role:** `if`; routes to reporting or setup branch.
- **Configuration choices:** checks whether field `"Day of week"` exists in the current item.
- **Interpretation:** scheduled trigger data includes that field; manual trigger data does not.
- **Input/Output:**
  - true branch → reporting data collection nodes
  - false branch → **List data tables**
- **Version notes:** typeVersion `2.3`.
- **Edge cases:** brittle trigger-shape detection; if trigger payload format changes, routing may fail.

---

## 2.2 Scheduled Reporting: Collect TMetric Data and Build Per-User Repository

### Overview
This block gathers all month-relevant TMetric data required for later user-level calculations. It combines users with their work schedules and grouped time-off requests, producing a normalized repository object per user.

### Nodes Involved
- Get TMetric Users
- Split Users Data
- Get TMetric User Work Schedule
- Rename Keys
- Get TMetric Time Off Requests
- Edit Fields
- Group Time off requests
- Remove Duplicates
- Add User Work Schedule
- Add Time off requests
- User Repo

### Node Details

#### Get TMetric Users
- **Type and role:** `httpRequest`; fetches TMetric reporting/filter payload containing users.
- **Configuration choices:**
  - GET to `.../accounts/{tmAccountId}/reports/projects/filter`
  - Header auth credential: TMetric API key
  - retries enabled: 5 attempts, 5 seconds apart
- **Key expressions:**
  - `{{$('Globals').item.json.apiBaseUrl}}`
  - `{{$('Globals').item.json["tmAccountId"]}}`
- **Input/Output:** from **Is running Setup?** true branch; outputs to **Split Users Data**
- **Version notes:** typeVersion `3`.
- **Edge cases:** invalid API key, missing account ID, endpoint response shape changes, 429/5xx responses.

#### Split Users Data
- **Type and role:** `itemLists`; splits the `users` array into one item per user.
- **Configuration:** `fieldToSplitOut = users`
- **Input/Output:** from **Get TMetric Users**; outputs to **Add User Work Schedule**
- **Version notes:** typeVersion `3`.
- **Edge cases:** missing `users` field, empty array, wrong response schema.

#### Get TMetric User Work Schedule
- **Type and role:** `httpRequest`; retrieves account work schedule from month start to current date.
- **Configuration choices:**
  - GET `.../accounts/{tmAccountId}/schedule`
  - query params:
    - `startDate = beginningOfTheMonth`
    - `endDate = fullEndDate`
  - Header auth, retries enabled
- **Input/Output:** from **Is running Setup?** true branch; outputs to **Rename Keys**
- **Version notes:** typeVersion `3`.
- **Edge cases:** auth failures, empty schedule, holidays/non-working-day shape differences.

#### Rename Keys
- **Type and role:** `renameKeys`; standardizes the schedule payload.
- **Configuration:** renames `days` → `workSchedule`
- **Input/Output:** from **Get TMetric User Work Schedule**; outputs to **Add User Work Schedule** input 1
- **Version notes:** typeVersion `1`.
- **Edge cases:** if `days` is absent, merge later may fail or produce incomplete data.

#### Get TMetric Time Off Requests
- **Type and role:** `httpRequest`; fetches time-off requests for the month.
- **Configuration choices:**
  - GET `.../accounts/{tmAccountId}/timeoff/requests`
  - query params:
    - `startDate = beginningOfTheMonth`
    - `endDate = endOfTheMonth`
  - Header auth, retries enabled
- **Input/Output:** from **Is running Setup?** true branch; outputs to **Edit Fields**
- **Version notes:** typeVersion `3`.
- **Edge cases:** endpoint may return all statuses; approval filtering is deferred; schema changes may break field extraction.

#### Edit Fields
- **Type and role:** `set`; flattens time-off request structure.
- **Configuration choices:** creates:
  - `requestId`
  - `requesterId`
  - `startDate`
  - `endDate`
  - `approved` = `status === 'Approved'`
- **Input/Output:** from **Get TMetric Time Off Requests**; outputs to **Group Time off requests**
- **Version notes:** typeVersion `3.4`.
- **Edge cases:** missing `requester.id`, null dates, status naming changes.

#### Group Time off requests
- **Type and role:** `set`; constructs grouped requests per requester.
- **Configuration choices:**
  - `timeOffRequests` = all input requests whose `requesterId` equals current item requesterId
  - `user = { id: requesterId }`
- **Key expressions:** uses `$input.all()` to aggregate all time-off items in the node run.
- **Input/Output:** from **Edit Fields**; outputs to **Remove Duplicates**
- **Version notes:** typeVersion `3.4`.
- **Edge cases:** includes all matching requests, regardless of `approved`; if approval filtering is desired, this block currently does not enforce it.

#### Remove Duplicates
- **Type and role:** `removeDuplicates`; keeps one grouped item per `user.id`.
- **Configuration:** compare selected field `user.id`
- **Input/Output:** from **Group Time off requests**; outputs to **Add Time off requests** input 1
- **Version notes:** typeVersion `2`.
- **Edge cases:** if `user.id` is missing, duplicate removal may be inconsistent.

#### Add User Work Schedule
- **Type and role:** `merge`; enriches each split user with schedule data.
- **Configuration choices:**
  - mode: combine
  - joinMode: `enrichInput1`
  - merge by `id` from user item and `user.id` from schedule payload
- **Input/Output:**
  - input 0 from **Split Users Data**
  - input 1 from **Rename Keys**
  - output to **Add Time off requests**
- **Version notes:** typeVersion `3.2`.
- **Edge cases:** if user IDs do not align exactly, user proceeds without schedule enrichment.

#### Add Time off requests
- **Type and role:** `merge`; enriches user+schedule items with grouped time off requests.
- **Configuration choices:**
  - mode: combine
  - joinMode: `enrichInput1`
  - merge by `user.id` on both sides
- **Input/Output:**
  - input 0 from **Add User Work Schedule**
  - input 1 from **Remove Duplicates**
  - output to **User Repo**
- **Version notes:** typeVersion `3.2`.
- **Edge cases:** users without time-off records continue, but later node must normalize empties.

#### User Repo
- **Type and role:** `set`; creates a normalized user repository object for downstream processing.
- **Configuration choices:** raw JSON output:
  - `user`
  - `workSchedule` or `[]`
  - `timeOffRequests` or `[]`
- **Input/Output:** from **Add Time off requests**; outputs to **Process Each User Separately**
- **Version notes:** typeVersion `3.4`.
- **Edge cases:** depends on `.isNotEmpty()` expressions; expression support should be verified in current n8n version.

---

## 2.3 Per-User Time Entry Retrieval, Aggregation, and Metrics Computation

### Overview
This block iterates through each user repository, fetches their time entries, aggregates the entries, combines them with the repository data, and computes personalized metrics such as hours behind schedule and project summaries.

### Nodes Involved
- Process Each User Separately
- Get Tmetric Time Entries per user
- Filter out running tasks
- Extract data from Time Entries
- Merge user info with time entries
- Wait
- Calculate data for personalized message

### Node Details

#### Process Each User Separately
- **Type and role:** `splitInBatches`; loops through users one at a time.
- **Configuration:** default batch behavior.
- **Input/Output:**
  - main output 0 triggers:
    - **Get row(s)**
    - **Merge** input 1
  - loop output 1 triggers:
    - **Get Tmetric Time Entries per user**
    - **Merge user info with time entries** input 0
  - receives loop-back from **Calculate data for personalized message**
- **Version notes:** typeVersion `3`.
- **Edge cases:** if loop termination is misconfigured, infinite loops are possible; current wiring appears intentional.

#### Get Tmetric Time Entries per user
- **Type and role:** `httpRequest`; fetches time entries for the current user from month start to current date.
- **Configuration choices:**
  - GET `.../accounts/{tmAccountId}/timeentries`
  - query params:
    - `userId = $json.user.id`
    - `startDate = beginningOfTheMonth`
    - `endDate = fullEndDate`
  - always output data enabled
- **Input/Output:** from **Process Each User Separately** loop output; outputs to **Filter out running tasks**
- **Version notes:** typeVersion `4.3`.
- **Edge cases:** empty results, invalid user IDs, API pagination assumptions, rate limits.

#### Filter out running tasks
- **Type and role:** `filter`; keeps only entries where `endTime` exists.
- **Configuration choices:** condition `endTime exists`
- **Purpose:** excludes currently running timers because they lack a completed duration.
- **Input/Output:** from **Get Tmetric Time Entries per user**; outputs to **Extract data from Time Entries**
- **Version notes:** typeVersion `2.3`.
- **Edge cases:** with `alwaysOutputData`, empty item flows may continue; duration calculations later must tolerate empty arrays.

#### Extract data from Time Entries
- **Type and role:** `set`; aggregates all filtered entry items into a single `time_entries` array.
- **Configuration choices:** raw JSON output with mapped fields:
  - `project`
  - `task` or fallback to `note`
  - `startTime`
  - `endTime`
  - `timeSpentInSeconds`
  - `note`
  - `isBillable`
  - `isInvoiced`
- **Key expressions:** uses `$input.all()` and computes `diffTo(..., 'seconds')`
- **Input/Output:** from **Filter out running tasks**; outputs to **Merge user info with time entries** input 1
- **Version notes:** typeVersion `3.4`, `executeOnce=true`
- **Edge cases:** the expression sets `task` to `json.task.name` rather than full task object, while later code expects richer task structure; this mismatch may affect task-level grouping.

#### Merge user info with time entries
- **Type and role:** `merge`; combines current user repository with extracted time entries by position.
- **Configuration:** `combineByPosition`
- **Input/Output:**
  - input 0 from **Process Each User Separately**
  - input 1 from **Extract data from Time Entries**
  - output to **Wait**
- **Version notes:** typeVersion `3.2`.
- **Edge cases:** relies on positional alignment; if either side emits unexpected item counts, merges can misalign.

#### Wait
- **Type and role:** `wait`; throttles the loop.
- **Configuration:** no explicit delay set in JSON, so behavior should be checked in UI.
- **Input/Output:** from **Merge user info with time entries**; outputs to **Calculate data for personalized message**
- **Version notes:** typeVersion `1.1`.
- **Edge cases:** if no wait duration is configured, this may simply resume immediately; if configured externally, it may pause execution significantly.

#### Calculate data for personalized message
- **Type and role:** `code`; calculates worked time, expected time, behind-schedule percentage, and per-project summaries.
- **Configuration choices:** JavaScript code node, execute once.
- **Key logic:**
  - loads project metadata from `$('Percentages of budget spent').all()`
  - sums `time_entries`
  - normalizes time-off request dates
  - computes expected schedule seconds
  - computes missing time and percentage behind schedule
  - groups entries by project and then by task/note
  - attaches `computed.projects`
- **Input/Output:** from **Wait**; outputs back to **Process Each User Separately**
- **Version notes:** typeVersion `2`.
- **Edge cases / important issues:**
  - **Broken reference risk:** node references `Percentages of budget spent`, but that node does not exist in this workflow JSON.
  - **Time-off exclusion bug risk:** code uses `normalizedTimeOffRequests.includes((req) => req.date === curr.date)`, which will always be false because `includes` checks values, not predicates.
  - **Task shape mismatch:** code expects `entry.task.id`, but upstream extraction may reduce task to a name string.
  - Division by zero possible if expected schedule is 0.
  - Date parsing/timezone inconsistencies may affect durations.

---

## 2.4 Slack Mapping Lookup and Report Delivery

### Overview
This block loads the TMetric-to-Slack mapping, joins it with the current user’s computed report, and sends a personalized Slack message.

### Nodes Involved
- Get row(s)
- Merge
- Send a message

### Node Details

#### Get row(s)
- **Type and role:** `dataTable`; retrieves all rows from the mapping table.
- **Configuration choices:**
  - operation: `get`
  - return all rows
  - table selected by name from **Globals**
  - execute once
- **Input/Output:** from **Process Each User Separately** main output 0; outputs to **Merge**
- **Version notes:** typeVersion `1.1`.
- **Edge cases:** missing table, empty mapping, permission issues in Data Tables feature.

#### Merge
- **Type and role:** `merge`; enriches Data Table rows with computed user data.
- **Configuration choices:**
  - mode: combine
  - joinMode: `enrichInput2`
  - fields:
    - `tmetric_user_id` from Data Table
    - `user.id` from computed user item
- **Input/Output:**
  - input 0 from **Get row(s)**
  - input 1 from **Process Each User Separately**
  - output to **Send a message**
- **Version notes:** typeVersion `3.2`.
- **Edge cases:** if no Slack mapping exists for a TMetric user, that user will not receive a message.

#### Send a message
- **Type and role:** `slack`; sends a direct Slack message using Block Kit.
- **Configuration choices:**
  - target user ID = `{{$json.slack_user_id}}`
  - message type = block
  - JSON block content includes:
    - greeting header
    - weekly report text
    - status button based on `percentageBehindSchedule`
    - project summary sections
  - Slack OAuth2 authentication
- **Key expressions:**
  - uses `slack_display_name`
  - compares `computed.percentageBehindSchedule` to `Globals['Allow hours missing percentage']`
  - formats project durations
- **Input/Output:** from **Merge**; terminal node on reporting branch.
- **Version notes:** typeVersion `2.4`.
- **Edge cases:**
  - invalid Slack user ID
  - bot lacks permission to DM users
  - malformed Block Kit JSON if data contains special characters
  - empty projects array may produce malformed block formatting depending on expression output

---

## 2.5 Setup Branch: Ensure Mapping Table Exists

### Overview
This block checks whether the required Data Table already exists and creates it if necessary.

### Nodes Involved
- List data tables
- Does Data table already exist?
- Create a data table

### Node Details

#### List data tables
- **Type and role:** `dataTable`; lists available Data Tables.
- **Configuration:** resource `table`
- **Input/Output:** from **Is running Setup?** false branch; outputs to **Does Data table already exist?**
- **Version notes:** typeVersion `1.1`.
- **Edge cases:** feature unavailable in self-hosted plan/version, permission issues.

#### Does Data table already exist?
- **Type and role:** `if`; checks whether a listed table name matches the configured name.
- **Configuration choices:** compares `$json.name` to `Globals.tmetricToSlackUserDataTableName`
- **Input/Output:**
  - true branch → **Get TMetric Users for user mapping**
  - false branch → **Create a data table**
- **Version notes:** typeVersion `2.3`.
- **Edge cases:** this logic operates item-by-item over table list; behavior should be validated when multiple tables exist, because false branch may trigger for non-matching rows before a match appears.

#### Create a data table
- **Type and role:** `dataTable`; creates the Slack/TMetric mapping table.
- **Configuration choices:**
  - table name from **Globals**
  - columns:
    - `tmetric_user_id` number
    - `slack_user_id`
    - `slack_display_name`
    - `slack_name`
- **Input/Output:** from false branch of **Does Data table already exist?**; outputs to **Get TMetric Users for user mapping**
- **Version notes:** typeVersion `1.1`.
- **Edge cases:** duplicate table names, table creation permissions, schema changes later causing upsert mismatch.

---

## 2.6 Setup Branch: Collect Users and Ask Slack Users to Map Themselves

### Overview
This block retrieves TMetric users and Slack users, filters valid Slack accounts, loops through Slack users, and sends each a form asking them to choose their TMetric account.

### Nodes Involved
- Get TMetric Users for user mapping
- Get many users
- Filter out users
- Send form for each user
- Send message and wait for response

### Node Details

#### Get TMetric Users for user mapping
- **Type and role:** `httpRequest`; retrieves TMetric users for the setup form options.
- **Configuration:** same TMetric endpoint and auth approach as reporting user fetch.
- **Input/Output:** from **Create a data table** or true branch of **Does Data table already exist?**; outputs to **Get many users**
- **Version notes:** typeVersion `3`.
- **Edge cases:** same as primary TMetric user fetch.

#### Get many users
- **Type and role:** `slack`; gets all Slack users in the workspace.
- **Configuration choices:**
  - resource `user`
  - operation `getAll`
  - `returnAll = true`
  - Slack OAuth2 auth
- **Input/Output:** from **Get TMetric Users for user mapping**; outputs to **Filter out users**
- **Version notes:** typeVersion `2.4`.
- **Edge cases:** workspace size, pagination/load, missing scopes such as user read permissions.

#### Filter out users
- **Type and role:** `filter`; removes invalid Slack accounts from setup targeting.
- **Configuration choices:** keep only users where:
  - `is_bot = false`
  - `id != USLACKBOT`
  - `deleted = false`
- **Input/Output:** from **Get many users**; outputs to **Send form for each user**
- **Version notes:** typeVersion `2.3`.
- **Edge cases:** guests and restricted users are still included unless filtered elsewhere.

#### Send form for each user
- **Type and role:** `splitInBatches`; iterates through Slack users.
- **Configuration:** default batching.
- **Input/Output:**
  - loop output 1 → **Send message and wait for response**
  - input receives loop-back from **Upsert row(s)**
- **Version notes:** typeVersion `3`.
- **Edge cases:** as with any loop, batch progression depends on correct loop-back.

#### Send message and wait for response
- **Type and role:** `slack`; sends an interactive message and waits for submitted form data.
- **Configuration choices:**
  - sends to a specific channel, not DM
  - message pings user `<@id>` and asks them to select their TMetric account
  - uses custom form with dropdown options built from `Get TMetric Users for user mapping`
  - response type `customForm`
- **Key expressions:**
  - dropdown values generated from `users.map(user => ({ option: `${user.name} <<${user.id}>>` }))`
- **Input/Output:** from **Send form for each user**; outputs to **Upsert row(s)**
- **Version notes:** typeVersion `2.4`.
- **Edge cases:**
  - channel ID must be configured
  - Slack app must support interactive forms
  - if user submits no selection, downstream regex extraction may fail
  - note typo in text: “anythging”

---

## 2.7 Setup Branch: Store User Mapping

### Overview
This block stores each Slack user’s selected TMetric account in the Data Table, updating existing rows when the same Slack user responds again.

### Nodes Involved
- Upsert row(s)

### Node Details

#### Upsert row(s)
- **Type and role:** `dataTable`; inserts or updates mapping rows.
- **Configuration choices:**
  - target table by configured name
  - filters rows where `slack_user_id` equals current Slack user ID
  - values written:
    - `slack_user_id`
    - `tmetric_user_id` extracted from selected dropdown text via regex
    - `slack_display_name`
- **Key expressions:**
  - `$('Send form for each user').item.json.id`
  - regex extraction from `$json.data["TMetric account"]`
- **Input/Output:** from **Send message and wait for response**; loops back to **Send form for each user**
- **Version notes:** typeVersion `1.1`.
- **Edge cases:**
  - if user submits no TMetric account, `.match(/<<(\d+)>>/gm)[0]` will fail
  - schema includes `slack_name` column in table creation, but upsert does not populate it
  - type conversion may be needed if regex result remains a string

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Run Once a Week | scheduleTrigger | Scheduled weekly entry point for reporting |  | Globals | ## Set up Globals and check if workflow runs from schedule or click<br>Because we specify in "Globals" data that is required also in "Setup" we must check what invoked "If" node. Luckily, it's very easy, because "Run Once a Week" trigger returns data which we can check against in our "If" node |
| Setup | manualTrigger | Manual entry point for mapping setup |  | Globals | ## Run once before you run on schedule<br><br>This workflow requires N8N Data Table in order to work. Once you run "Setup" workflow will check if required Data Table already exists, if no, it will create it.<br>Then, it will make API Call to Tmetric, retrieve users data, retrieve Slack users and filter out accounts that are invalid.<br><br>Then it will send a message to the channel, pinging specific users to fill out the form where they are requested to select their Tmetric account.<br><br>This is needed in order to be able to send later personalized private messages directly to the users<br><br>### Run again<br><br>If new user has been added to Tmetric or Slack, or somebody made mistake, you can safely run this step again. It won't recreate the table but send a form again to all users<br><br>### Important<br><br>If user shouldn't be notified they should just submit the form without selecting any value |
| Globals | set | Stores shared constants and date boundaries | Run Once a Week, Setup | Is running Setup? | ## Set up Globals and check if workflow runs from schedule or click<br>Because we specify in "Globals" data that is required also in "Setup" we must check what invoked "If" node. Luckily, it's very easy, because "Run Once a Week" trigger returns data which we can check against in our "If" node |
| Is running Setup? | if | Routes to reporting or setup branch | Globals | Get TMetric Users, Get TMetric User Work Schedule, Get TMetric Time Off Requests, List data tables | ## Set up Globals and check if workflow runs from schedule or click<br>Because we specify in "Globals" data that is required also in "Setup" we must check what invoked "If" node. Luckily, it's very easy, because "Run Once a Week" trigger returns data which we can check against in our "If" node |
| Get TMetric Users | httpRequest | Fetches TMetric users for reporting | Is running Setup? | Split Users Data | ## Step 1:  Collecting the data from Tmetric and process it<br>In this step we use custom API calls to Tmetric API to collect are necessary data. Then we match it to the correct user, assign it to make easier to use later.<br><br>For Work Schedule we get data from beginning of the month up until now, so we can calculate missing time correctly<br>For Projects Data we fetch entire month so we know how much of the budget was spent<br><br>In the end we use set to create cohesieve repository that will be easy to work with<br><br>If you want to learn more about Tmetric API check their docs [here](https://app.tmetric.com/api-docs/) |
| Split Users Data | itemLists | Splits TMetric users array into items | Get TMetric Users | Add User Work Schedule | ## Step 1:  Collecting the data from Tmetric and process it<br>In this step we use custom API calls to Tmetric API to collect are necessary data. Then we match it to the correct user, assign it to make easier to use later.<br><br>For Work Schedule we get data from beginning of the month up until now, so we can calculate missing time correctly<br>For Projects Data we fetch entire month so we know how much of the budget was spent<br><br>In the end we use set to create cohesieve repository that will be easy to work with<br><br>If you want to learn more about Tmetric API check their docs [here](https://app.tmetric.com/api-docs/) |
| Get TMetric User Work Schedule | httpRequest | Fetches account work schedule | Is running Setup? | Rename Keys | ## Step 1:  Collecting the data from Tmetric and process it<br>In this step we use custom API calls to Tmetric API to collect are necessary data. Then we match it to the correct user, assign it to make easier to use later.<br><br>For Work Schedule we get data from beginning of the month up until now, so we can calculate missing time correctly<br>For Projects Data we fetch entire month so we know how much of the budget was spent<br><br>In the end we use set to create cohesieve repository that will be easy to work with<br><br>If you want to learn more about Tmetric API check their docs [here](https://app.tmetric.com/api-docs/) |
| Rename Keys | renameKeys | Renames schedule field for consistency | Get TMetric User Work Schedule | Add User Work Schedule | ## Step 1:  Collecting the data from Tmetric and process it<br>In this step we use custom API calls to Tmetric API to collect are necessary data. Then we match it to the correct user, assign it to make easier to use later.<br><br>For Work Schedule we get data from beginning of the month up until now, so we can calculate missing time correctly<br>For Projects Data we fetch entire month so we know how much of the budget was spent<br><br>In the end we use set to create cohesieve repository that will be easy to work with<br><br>If you want to learn more about Tmetric API check their docs [here](https://app.tmetric.com/api-docs/) |
| Get TMetric Time Off Requests | httpRequest | Fetches monthly time-off requests | Is running Setup? | Edit Fields | ## Step 1:  Collecting the data from Tmetric and process it<br>In this step we use custom API calls to Tmetric API to collect are necessary data. Then we match it to the correct user, assign it to make easier to use later.<br><br>For Work Schedule we get data from beginning of the month up until now, so we can calculate missing time correctly<br>For Projects Data we fetch entire month so we know how much of the budget was spent<br><br>In the end we use set to create cohesieve repository that will be easy to work with<br><br>If you want to learn more about Tmetric API check their docs [here](https://app.tmetric.com/api-docs/) |
| Edit Fields | set | Flattens time-off request fields | Get TMetric Time Off Requests | Group Time off requests | ## Step 1:  Collecting the data from Tmetric and process it<br>In this step we use custom API calls to Tmetric API to collect are necessary data. Then we match it to the correct user, assign it to make easier to use later.<br><br>For Work Schedule we get data from beginning of the month up until now, so we can calculate missing time correctly<br>For Projects Data we fetch entire month so we know how much of the budget was spent<br><br>In the end we use set to create cohesieve repository that will be easy to work with<br><br>If you want to learn more about Tmetric API check their docs [here](https://app.tmetric.com/api-docs/) |
| Group Time off requests | set | Groups time-off requests by requester | Edit Fields | Remove Duplicates | ## Step 1:  Collecting the data from Tmetric and process it<br>In this step we use custom API calls to Tmetric API to collect are necessary data. Then we match it to the correct user, assign it to make easier to use later.<br><br>For Work Schedule we get data from beginning of the month up until now, so we can calculate missing time correctly<br>For Projects Data we fetch entire month so we know how much of the budget was spent<br><br>In the end we use set to create cohesieve repository that will be easy to work with<br><br>If you want to learn more about Tmetric API check their docs [here](https://app.tmetric.com/api-docs/) |
| Remove Duplicates | removeDuplicates | Keeps one grouped time-off item per user | Group Time off requests | Add Time off requests | ## Step 1:  Collecting the data from Tmetric and process it<br>In this step we use custom API calls to Tmetric API to collect are necessary data. Then we match it to the correct user, assign it to make easier to use later.<br><br>For Work Schedule we get data from beginning of the month up until now, so we can calculate missing time correctly<br>For Projects Data we fetch entire month so we know how much of the budget was spent<br><br>In the end we use set to create cohesieve repository that will be easy to work with<br><br>If you want to learn more about Tmetric API check their docs [here](https://app.tmetric.com/api-docs/) |
| Add User Work Schedule | merge | Enriches users with work schedule | Split Users Data, Rename Keys | Add Time off requests | ## Step 1:  Collecting the data from Tmetric and process it<br>In this step we use custom API calls to Tmetric API to collect are necessary data. Then we match it to the correct user, assign it to make easier to use later.<br><br>For Work Schedule we get data from beginning of the month up until now, so we can calculate missing time correctly<br>For Projects Data we fetch entire month so we know how much of the budget was spent<br><br>In the end we use set to create cohesieve repository that will be easy to work with<br><br>If you want to learn more about Tmetric API check their docs [here](https://app.tmetric.com/api-docs/) |
| Add Time off requests | merge | Enriches users with grouped time-off requests | Add User Work Schedule, Remove Duplicates | User Repo | ## Step 1:  Collecting the data from Tmetric and process it<br>In this step we use custom API calls to Tmetric API to collect are necessary data. Then we match it to the correct user, assign it to make easier to use later.<br><br>For Work Schedule we get data from beginning of the month up until now, so we can calculate missing time correctly<br>For Projects Data we fetch entire month so we know how much of the budget was spent<br><br>In the end we use set to create cohesieve repository that will be easy to work with<br><br>If you want to learn more about Tmetric API check their docs [here](https://app.tmetric.com/api-docs/) |
| User Repo | set | Normalizes per-user repository object | Add Time off requests | Process Each User Separately | ## Step 1:  Collecting the data from Tmetric and process it<br>In this step we use custom API calls to Tmetric API to collect are necessary data. Then we match it to the correct user, assign it to make easier to use later.<br><br>For Work Schedule we get data from beginning of the month up until now, so we can calculate missing time correctly<br>For Projects Data we fetch entire month so we know how much of the budget was spent<br><br>In the end we use set to create cohesieve repository that will be easy to work with<br><br>If you want to learn more about Tmetric API check their docs [here](https://app.tmetric.com/api-docs/) |
| Process Each User Separately | splitInBatches | Iterates through users for user-specific processing | User Repo, Calculate data for personalized message | Get row(s), Merge, Get Tmetric Time Entries per user, Merge user info with time entries | ## Step 2: Process each user, fetch more user specific data, and generate data for personalized feedback<br>In this step we have to go back to Tmetric API. Using previously obtained User ID, we are obtaining users Time Entries without currently running tasks that are later used to calculate how much time they in fact spent working. <br>We also use Work Schedule and Time Off Requests to see how much time user should actually have spent. Then, we convert it to percentages to see if user met our criteria<br><br>We use wait node to take a short break between successive calls so we don't hit rate limits. |
| Get Tmetric Time Entries per user | httpRequest | Fetches current user time entries | Process Each User Separately | Filter out running tasks | ## Step 2: Process each user, fetch more user specific data, and generate data for personalized feedback<br>In this step we have to go back to Tmetric API. Using previously obtained User ID, we are obtaining users Time Entries without currently running tasks that are later used to calculate how much time they in fact spent working. <br>We also use Work Schedule and Time Off Requests to see how much time user should actually have spent. Then, we convert it to percentages to see if user met our criteria<br><br>We use wait node to take a short break between successive calls so we don't hit rate limits. |
| Filter out running tasks | filter | Removes unfinished/running time entries | Get Tmetric Time Entries per user | Extract data from Time Entries | ## Step 2: Process each user, fetch more user specific data, and generate data for personalized feedback<br>In this step we have to go back to Tmetric API. Using previously obtained User ID, we are obtaining users Time Entries without currently running tasks that are later used to calculate how much time they in fact spent working. <br>We also use Work Schedule and Time Off Requests to see how much time user should actually have spent. Then, we convert it to percentages to see if user met our criteria<br><br>We use wait node to take a short break between successive calls so we don't hit rate limits. |
| Extract data from Time Entries | set | Aggregates filtered entries into a single array | Filter out running tasks | Merge user info with time entries | ## Step 2: Process each user, fetch more user specific data, and generate data for personalized feedback<br>In this step we have to go back to Tmetric API. Using previously obtained User ID, we are obtaining users Time Entries without currently running tasks that are later used to calculate how much time they in fact spent working. <br>We also use Work Schedule and Time Off Requests to see how much time user should actually have spent. Then, we convert it to percentages to see if user met our criteria<br><br>We use wait node to take a short break between successive calls so we don't hit rate limits. |
| Merge user info with time entries | merge | Combines repository and time entries | Process Each User Separately, Extract data from Time Entries | Wait | ## Step 2: Process each user, fetch more user specific data, and generate data for personalized feedback<br>In this step we have to go back to Tmetric API. Using previously obtained User ID, we are obtaining users Time Entries without currently running tasks that are later used to calculate how much time they in fact spent working. <br>We also use Work Schedule and Time Off Requests to see how much time user should actually have spent. Then, we convert it to percentages to see if user met our criteria<br><br>We use wait node to take a short break between successive calls so we don't hit rate limits. |
| Wait | wait | Adds pause between per-user iterations | Merge user info with time entries | Calculate data for personalized message | ## Step 2: Process each user, fetch more user specific data, and generate data for personalized feedback<br>In this step we have to go back to Tmetric API. Using previously obtained User ID, we are obtaining users Time Entries without currently running tasks that are later used to calculate how much time they in fact spent working. <br>We also use Work Schedule and Time Off Requests to see how much time user should actually have spent. Then, we convert it to percentages to see if user met our criteria<br><br>We use wait node to take a short break between successive calls so we don't hit rate limits. |
| Calculate data for personalized message | code | Computes user metrics and project summaries | Wait | Process Each User Separately | ## Step 2: Process each user, fetch more user specific data, and generate data for personalized feedback<br>In this step we have to go back to Tmetric API. Using previously obtained User ID, we are obtaining users Time Entries without currently running tasks that are later used to calculate how much time they in fact spent working. <br>We also use Work Schedule and Time Off Requests to see how much time user should actually have spent. Then, we convert it to percentages to see if user met our criteria<br><br>We use wait node to take a short break between successive calls so we don't hit rate limits. |
| Get row(s) | dataTable | Loads Slack/TMetric mapping rows | Process Each User Separately | Merge | ## Step 3: Send raport<br>In this step we use N8N Data Table we created previously and look for users Slack ID so we can send them message directly.<br><br>Message is built using Slack Block Builder Kit available under [this link](https://app.slack.com/block-kit-builder/). With it's help we can create styled messages that are not boring. |
| Merge | merge | Joins computed user report with Slack mapping | Get row(s), Process Each User Separately | Send a message | ## Step 3: Send raport<br>In this step we use N8N Data Table we created previously and look for users Slack ID so we can send them message directly.<br><br>Message is built using Slack Block Builder Kit available under [this link](https://app.slack.com/block-kit-builder/). With it's help we can create styled messages that are not boring. |
| Send a message | slack | Sends personalized Slack report | Merge |  | ## Step 3: Send raport<br>In this step we use N8N Data Table we created previously and look for users Slack ID so we can send them message directly.<br><br>Message is built using Slack Block Builder Kit available under [this link](https://app.slack.com/block-kit-builder/). With it's help we can create styled messages that are not boring. |
| List data tables | dataTable | Lists available Data Tables | Is running Setup? | Does Data table already exist? | ## Run once before you run on schedule<br><br>This workflow requires N8N Data Table in order to work. Once you run "Setup" workflow will check if required Data Table already exists, if no, it will create it.<br>Then, it will make API Call to Tmetric, retrieve users data, retrieve Slack users and filter out accounts that are invalid.<br><br>Then it will send a message to the channel, pinging specific users to fill out the form where they are requested to select their Tmetric account.<br><br>This is needed in order to be able to send later personalized private messages directly to the users<br><br>### Run again<br><br>If new user has been added to Tmetric or Slack, or somebody made mistake, you can safely run this step again. It won't recreate the table but send a form again to all users<br><br>### Important<br><br>If user shouldn't be notified they should just submit the form without selecting any value |
| Does Data table already exist? | if | Checks whether mapping table exists | List data tables | Get TMetric Users for user mapping, Create a data table | ## Run once before you run on schedule<br><br>This workflow requires N8N Data Table in order to work. Once you run "Setup" workflow will check if required Data Table already exists, if no, it will create it.<br>Then, it will make API Call to Tmetric, retrieve users data, retrieve Slack users and filter out accounts that are invalid.<br><br>Then it will send a message to the channel, pinging specific users to fill out the form where they are requested to select their Tmetric account.<br><br>This is needed in order to be able to send later personalized private messages directly to the users<br><br>### Run again<br><br>If new user has been added to Tmetric or Slack, or somebody made mistake, you can safely run this step again. It won't recreate the table but send a form again to all users<br><br>### Important<br><br>If user shouldn't be notified they should just submit the form without selecting any value |
| Create a data table | dataTable | Creates mapping table if missing | Does Data table already exist? | Get TMetric Users for user mapping | ## Run once before you run on schedule<br><br>This workflow requires N8N Data Table in order to work. Once you run "Setup" workflow will check if required Data Table already exists, if no, it will create it.<br>Then, it will make API Call to Tmetric, retrieve users data, retrieve Slack users and filter out accounts that are invalid.<br><br>Then it will send a message to the channel, pinging specific users to fill out the form where they are requested to select their Tmetric account.<br><br>This is needed in order to be able to send later personalized private messages directly to the users<br><br>### Run again<br><br>If new user has been added to Tmetric or Slack, or somebody made mistake, you can safely run this step again. It won't recreate the table but send a form again to all users<br><br>### Important<br><br>If user shouldn't be notified they should just submit the form without selecting any value |
| Get TMetric Users for user mapping | httpRequest | Fetches TMetric users for setup form options | Does Data table already exist?, Create a data table | Get many users | ## Run once before you run on schedule<br><br>This workflow requires N8N Data Table in order to work. Once you run "Setup" workflow will check if required Data Table already exists, if no, it will create it.<br>Then, it will make API Call to Tmetric, retrieve users data, retrieve Slack users and filter out accounts that are invalid.<br><br>Then it will send a message to the channel, pinging specific users to fill out the form where they are requested to select their Tmetric account.<br><br>This is needed in order to be able to send later personalized private messages directly to the users<br><br>### Run again<br><br>If new user has been added to Tmetric or Slack, or somebody made mistake, you can safely run this step again. It won't recreate the table but send a form again to all users<br><br>### Important<br><br>If user shouldn't be notified they should just submit the form without selecting any value |
| Get many users | slack | Retrieves all Slack workspace users | Get TMetric Users for user mapping | Filter out users | ## Run once before you run on schedule<br><br>This workflow requires N8N Data Table in order to work. Once you run "Setup" workflow will check if required Data Table already exists, if no, it will create it.<br>Then, it will make API Call to Tmetric, retrieve users data, retrieve Slack users and filter out accounts that are invalid.<br><br>Then it will send a message to the channel, pinging specific users to fill out the form where they are requested to select their Tmetric account.<br><br>This is needed in order to be able to send later personalized private messages directly to the users<br><br>### Run again<br><br>If new user has been added to Tmetric or Slack, or somebody made mistake, you can safely run this step again. It won't recreate the table but send a form again to all users<br><br>### Important<br><br>If user shouldn't be notified they should just submit the form without selecting any value |
| Filter out users | filter | Removes bots and deleted Slack accounts | Get many users | Send form for each user | ## Run once before you run on schedule<br><br>This workflow requires N8N Data Table in order to work. Once you run "Setup" workflow will check if required Data Table already exists, if no, it will create it.<br>Then, it will make API Call to Tmetric, retrieve users data, retrieve Slack users and filter out accounts that are invalid.<br><br>Then it will send a message to the channel, pinging specific users to fill out the form where they are requested to select their Tmetric account.<br><br>This is needed in order to be able to send later personalized private messages directly to the users<br><br>### Run again<br><br>If new user has been added to Tmetric or Slack, or somebody made mistake, you can safely run this step again. It won't recreate the table but send a form again to all users<br><br>### Important<br><br>If user shouldn't be notified they should just submit the form without selecting any value |
| Send form for each user | splitInBatches | Iterates through Slack users for form delivery | Filter out users, Upsert row(s) | Send message and wait for response | ## Run once before you run on schedule<br><br>This workflow requires N8N Data Table in order to work. Once you run "Setup" workflow will check if required Data Table already exists, if no, it will create it.<br>Then, it will make API Call to Tmetric, retrieve users data, retrieve Slack users and filter out accounts that are invalid.<br><br>Then it will send a message to the channel, pinging specific users to fill out the form where they are requested to select their Tmetric account.<br><br>This is needed in order to be able to send later personalized private messages directly to the users<br><br>### Run again<br><br>If new user has been added to Tmetric or Slack, or somebody made mistake, you can safely run this step again. It won't recreate the table but send a form again to all users<br><br>### Important<br><br>If user shouldn't be notified they should just submit the form without selecting any value |
| Send message and wait for response | slack | Sends interactive mapping form and waits for answer | Send form for each user | Upsert row(s) | ## Run once before you run on schedule<br><br>This workflow requires N8N Data Table in order to work. Once you run "Setup" workflow will check if required Data Table already exists, if no, it will create it.<br>Then, it will make API Call to Tmetric, retrieve users data, retrieve Slack users and filter out accounts that are invalid.<br><br>Then it will send a message to the channel, pinging specific users to fill out the form where they are requested to select their Tmetric account.<br><br>This is needed in order to be able to send later personalized private messages directly to the users<br><br>### Run again<br><br>If new user has been added to Tmetric or Slack, or somebody made mistake, you can safely run this step again. It won't recreate the table but send a form again to all users<br><br>### Important<br><br>If user shouldn't be notified they should just submit the form without selecting any value |
| Upsert row(s) | dataTable | Stores or updates Slack-to-TMetric mapping | Send message and wait for response | Send form for each user | ## Run once before you run on schedule<br><br>This workflow requires N8N Data Table in order to work. Once you run "Setup" workflow will check if required Data Table already exists, if no, it will create it.<br>Then, it will make API Call to Tmetric, retrieve users data, retrieve Slack users and filter out accounts that are invalid.<br><br>Then it will send a message to the channel, pinging specific users to fill out the form where they are requested to select their Tmetric account.<br><br>This is needed in order to be able to send later personalized private messages directly to the users<br><br>### Run again<br><br>If new user has been added to Tmetric or Slack, or somebody made mistake, you can safely run this step again. It won't recreate the table but send a form again to all users<br><br>### Important<br><br>If user shouldn't be notified they should just submit the form without selecting any value |
| Sticky Note1 | stickyNote | Documentation note |  |  | ## Step 1:  Collecting the data from Tmetric and process it<br>In this step we use custom API calls to Tmetric API to collect are necessary data. Then we match it to the correct user, assign it to make easier to use later.<br><br>For Work Schedule we get data from beginning of the month up until now, so we can calculate missing time correctly<br>For Projects Data we fetch entire month so we know how much of the budget was spent<br><br>In the end we use set to create cohesieve repository that will be easy to work with<br><br>If you want to learn more about Tmetric API check their docs [here](https://app.tmetric.com/api-docs/) |
| Sticky Note2 | stickyNote | Documentation note |  |  | ## Set up Globals and check if workflow runs from schedule or click<br>Because we specify in "Globals" data that is required also in "Setup" we must check what invoked "If" node. Luckily, it's very easy, because "Run Once a Week" trigger returns data which we can check against in our "If" node |
| Sticky Note4 | stickyNote | Documentation note |  |  | ## Step 2: Process each user, fetch more user specific data, and generate data for personalized feedback<br>In this step we have to go back to Tmetric API. Using previously obtained User ID, we are obtaining users Time Entries without currently running tasks that are later used to calculate how much time they in fact spent working. <br>We also use Work Schedule and Time Off Requests to see how much time user should actually have spent. Then, we convert it to percentages to see if user met our criteria<br><br>We use wait node to take a short break between successive calls so we don't hit rate limits. |
| Sticky Note5 | stickyNote | Documentation note |  |  | ## Step 3: Send raport<br>In this step we use N8N Data Table we created previously and look for users Slack ID so we can send them message directly.<br><br>Message is built using Slack Block Builder Kit available under [this link](https://app.slack.com/block-kit-builder/). With it's help we can create styled messages that are not boring. |
| Sticky Note7 | stickyNote | Documentation note |  |  | ## Run once before you run on schedule<br><br>This workflow requires N8N Data Table in order to work. Once you run "Setup" workflow will check if required Data Table already exists, if no, it will create it.<br>Then, it will make API Call to Tmetric, retrieve users data, retrieve Slack users and filter out accounts that are invalid.<br><br>Then it will send a message to the channel, pinging specific users to fill out the form where they are requested to select their Tmetric account.<br><br>This is needed in order to be able to send later personalized private messages directly to the users<br><br>### Run again<br><br>If new user has been added to Tmetric or Slack, or somebody made mistake, you can safely run this step again. It won't recreate the table but send a form again to all users<br><br>### Important<br><br>If user shouldn't be notified they should just submit the form without selecting any value |
| Sticky Note8 | stickyNote | Documentation note |  |  | ## TMetric to Slack employees time manager<br>### How it works<br>This workflow checks weekly if users meet their hourly quota at the end of trhe week.<br>It will help them keep track of how much they worked, on what projects, remind them if they forgot to put entries in the system<br><br>It leverages Tmetric API in order to obtain data, Slack, and N8N Data Tables<br><br>#### Create tables<br>1. We check if data table with given name already exists, if yes, we skip to the next part, if not, we create it<br>2. We get Tmetric and Slack users, filter out users that are bots or inactive<br>3. We sen<br><br>#### Run Once a Week<br>1. We colect data from TMetric using custom HTTP Requests, calculate percentage of project budget spent, getting users, their Work Schedules and Time Off Requests<br>2. We process each user individually, get their Time Entries, merge all data together into one cohesieve piece<br>3. We get users Slack id from N8N Data table we created during set up using their Tmetric id to send personalized message directly to them<br><br>###Setup<br>In order for workflow to work you will need to setup couple variables. Most likely you will be only interested in just 2 first, but you can also set name of the Data Table to prevent any problems<br><br>- Fill out "Globals" node<br>  - tmAccountId - You can find it in URL once you visit Tmetric, in this example it's all 0's: https://app.tmetric.com/#/tracker/000000/<br>  - Allow hours missing percentage - If users have more percentages of missing time spent on working they will get information about it<br>  - tmetricToSlackUserDataTableName - Name of the data table used across the project <br><br>- Auth<br>In order for this workflow to work you have to set up auth for HTTP Request Nodes. In order to do this, open one of these nodes and select:<br>Authentication > General Credential Type > Header Auth <br>And if you doesn't already have Tmetric credentials Create new of following shape:<br><br>Name:<br>Authorization<br>Value:<br>&lt;Your Tmetric API Key&gt;<br><br>You can learn how to get API key [here](https://app.tmetric.com/api-docs/)<br><br>### Customize<br>If you want to customize message that is sent on Slack edit "Send a message" node. You can also change there if message should be sent directly or to the channel<br><br><br><br><br><br>Need help? Contact us at developers@sailingbyte.com or at sailingbyte.com<br><br>Happy hacking! |
| 93ba4d70-3586-462a-a841-fdf5e735f55e? no, actual node is Get row(s) already covered |  |  |  |  |  |

> Note: The workflow contains sticky note nodes. They are included above because the requirement was not to skip any nodes.

---

# 4. Reproducing the Workflow from Scratch

## Prerequisites
1. Have an n8n instance with:
   - **Slack integration**
   - **Data Tables** feature enabled
   - outbound access to `https://app.tmetric.com/api/v3`
2. Obtain:
   - **TMetric API key**
   - **TMetric account ID**
   - **Slack OAuth2 credentials** with permissions to:
     - read users
     - send messages
     - use interactive forms / wait for responses
3. Create credentials in n8n:
   - **HTTP Header Auth**
     - Header name: `Authorization`
     - Header value: your TMetric API key
   - **Slack OAuth2**

## Build Steps

### A. Create entry points and globals
1. Add a **Schedule Trigger** node named **Run Once a Week**.
   - Configure weekly interval.
   - Set trigger day to `5` and hour to `16`.
   - Confirm weekday meaning in your environment.

2. Add a **Manual Trigger** node named **Setup**.

3. Add a **Set** node named **Globals**.
   - Add these fields:
     - `tmAccountId` as string, fill with your TMetric account ID
     - `Allow hours missing percentage` as number, default `10`
     - `tmetricToSlackUserDataTableName` as string, e.g. `Tmetric to Slack user map`
     - `apiBaseUrl` as string: `https://app.tmetric.com/api/v3`
     - `fullEndDate` as expression: current date formatted `yyyy-LL-dd`
     - `beginningOfTheMonth` as expression: first day of current month
     - `endOfTheMonth` as expression: last day of current month
   - Enable “include other fields”.

4. Connect **Run Once a Week → Globals** and **Setup → Globals**.

5. Add an **If** node named **Is running Setup?**
   - Condition: field `Day of week` exists.
   - Connect **Globals → Is running Setup?**

### B. Reporting branch: fetch TMetric users, schedules, and time off
6. Add an **HTTP Request** node named **Get TMetric Users**.
   - URL:
     `{{$('Globals').item.json.apiBaseUrl}}/accounts/{{ $('Globals').item.json["tmAccountId"] }}/reports/projects/filter`
   - Auth: Generic Credential Type → HTTP Header Auth
   - Credential: TMetric API key
   - Retry on fail: yes, 5 retries, 5000 ms

7. Add an **Item Lists** node named **Split Users Data**.
   - Operation: split out items from field `users`

8. Add an **HTTP Request** node named **Get TMetric User Work Schedule**.
   - URL:
     `{{$('Globals').item.json.apiBaseUrl}}/accounts/{{ $('Globals').item.json.tmAccountId }}/schedule`
   - Query params:
     - `startDate = {{$('Globals').item.json.beginningOfTheMonth}}`
     - `endDate = {{$('Globals').item.json.fullEndDate}}`
   - Same TMetric auth and retries

9. Add a **Rename Keys** node named **Rename Keys**.
   - Rename `days` to `workSchedule`

10. Add an **HTTP Request** node named **Get TMetric Time Off Requests**.
   - URL:
     `{{$('Globals').item.json.apiBaseUrl}}/accounts/{{ $('Globals').item.json["tmAccountId"] }}/timeoff/requests`
   - Query params:
     - `startDate = {{$('Globals').item.json.beginningOfTheMonth}}`
     - `endDate = {{$('Globals').item.json.endOfTheMonth}}`
   - Same TMetric auth and retries

11. Add a **Set** node named **Edit Fields**.
   - Create:
     - `requestId = {{$json.id}}`
     - `requesterId = {{$json.requester.id}}`
     - `startDate = {{$json.startDate}}`
     - `endDate = {{$json.endDate}}`
     - `approved = {{$json.status === 'Approved'}}`

12. Add a **Set** node named **Group Time off requests**.
   - Create:
     - `timeOffRequests = {{$input.all().map(({json})=>json).filter(item=>item.requesterId == $json.requesterId)}}`
     - `user` object with `id = {{$json.requesterId}}`

13. Add a **Remove Duplicates** node named **Remove Duplicates**.
   - Compare selected fields
   - Field: `user.id`

14. Add a **Merge** node named **Add User Work Schedule**.
   - Mode: Combine
   - Join mode: Enrich Input 1
   - Match:
     - input1 field `id`
     - input2 field `user.id`

15. Add a **Merge** node named **Add Time off requests**.
   - Mode: Combine
   - Join mode: Enrich Input 1
   - Match:
     - input1 field `user.id`
     - input2 field `user.id`

16. Add a **Set** node named **User Repo**.
   - Use raw JSON mode:
     ```json
     {
       "user": {{ $json.user }},
       "workSchedule": {{ $if($json.workSchedule.isNotEmpty(), $json.workSchedule, []) }},
       "timeOffRequests": {{ $if($json.timeOffRequests.isNotEmpty(), $json.timeOffRequests, []) }}
     }
     ```

17. Connect reporting branch:
   - **Is running Setup? true → Get TMetric Users**
   - **Is running Setup? true → Get TMetric User Work Schedule**
   - **Is running Setup? true → Get TMetric Time Off Requests**
   - **Get TMetric Users → Split Users Data**
   - **Get TMetric User Work Schedule → Rename Keys**
   - **Get TMetric Time Off Requests → Edit Fields → Group Time off requests → Remove Duplicates**
   - **Split Users Data → Add User Work Schedule input 1**
   - **Rename Keys → Add User Work Schedule input 2**
   - **Add User Work Schedule → Add Time off requests input 1**
   - **Remove Duplicates → Add Time off requests input 2**
   - **Add Time off requests → User Repo**

### C. Per-user loop
18. Add a **Split In Batches** node named **Process Each User Separately**.
   - Connect **User Repo → Process Each User Separately**

19. Add an **HTTP Request** node named **Get Tmetric Time Entries per user**.
   - URL:
     `{{$('Globals').item.json.apiBaseUrl}}/accounts/{{$('Globals').item.json.tmAccountId}}/timeentries`
   - Query params:
     - `userId = {{$json.user.id}}`
     - `startDate = {{$('Globals').item.json.beginningOfTheMonth}}`
     - `endDate = {{$('Globals').item.json.fullEndDate}}`
   - Auth: TMetric header auth
   - Enable `alwaysOutputData`

20. Add a **Filter** node named **Filter out running tasks**.
   - Keep items where `{{$json?.endTime}}` exists

21. Add a **Set** node named **Extract data from Time Entries**.
   - Raw JSON:
     ```json
     {
       "time_entries": {{ $input.all().map(({json})=>({
         "project": json?.project,
         "task": $if(json?.task, json?.task.name, json.note),
         "startTime": json.startTime,
         "endTime": json.endTime,
         "timeSpentInSeconds": json?.endTime?.toDateTime().diffTo(json?.startTime, 'seconds'),
         "note": json?.note,
         "isBillable": json?.isBillable,
         "isInvoiced": json?.isInvoiced
       })).filter(obj=>obj.project) }}
     }
     ```
   - Enable execute once

22. Add a **Merge** node named **Merge user info with time entries**.
   - Mode: Combine
   - Combine by position

23. Add a **Wait** node named **Wait**.
   - Configure a small delay if you want actual throttling.

24. Add a **Code** node named **Calculate data for personalized message**.
   - Paste the provided JavaScript logic if you want to mirror behavior exactly.
   - Recommended: fix these issues before production use:
     - remove or replace missing node reference `Percentages of budget spent`
     - replace incorrect `includes(predicate)` logic with `.some(req => req.date === curr.date)`
     - keep full task object in upstream extraction if task grouping by ID is required

25. Connect:
   - **Process Each User Separately loop output → Get Tmetric Time Entries per user**
   - **Get Tmetric Time Entries per user → Filter out running tasks → Extract data from Time Entries**
   - **Process Each User Separately loop output → Merge user info with time entries input 1**
   - **Extract data from Time Entries → Merge user info with time entries input 2**
   - **Merge user info with time entries → Wait → Calculate data for personalized message**
   - **Calculate data for personalized message → Process Each User Separately** for loop continuation

### D. Mapping lookup and Slack delivery
26. Add a **Data Table** node named **Get row(s)**.
   - Operation: Get
   - Return all: true
   - Table name: `{{$('Globals').first().json.tmetricToSlackUserDataTableName}}`
   - Enable execute once

27. Add a **Merge** node named **Merge**.
   - Mode: Combine
   - Join mode: Enrich Input 2
   - Match:
     - input1 field `tmetric_user_id`
     - input2 field `user.id`

28. Add a **Slack** node named **Send a message**.
   - Authentication: Slack OAuth2
   - Select: user
   - User ID expression: `{{$json.slack_user_id}}`
   - Message type: block
   - Paste/adapt the Block Kit JSON from the workflow.
   - Ensure expressions for:
     - `slack_display_name`
     - `computed.percentageBehindSchedule`
     - `computed.projects`

29. Connect:
   - **Process Each User Separately main output → Get row(s)**
   - **Process Each User Separately main output → Merge input 2**
   - **Get row(s) → Merge input 1**
   - **Merge → Send a message**

### E. Setup branch: create mapping table if missing
30. Add a **Data Table** node named **List data tables**.
   - Resource: table

31. Add an **If** node named **Does Data table already exist?**
   - Compare `{{$json.name}}` equals `{{$('Globals').first().json.tmetricToSlackUserDataTableName}}`

32. Add a **Data Table** node named **Create a data table**.
   - Operation: create table
   - Table name from **Globals**
   - Columns:
     - `tmetric_user_id` number
     - `slack_user_id`
     - `slack_display_name`
     - `slack_name`

33. Connect:
   - **Is running Setup? false → List data tables → Does Data table already exist?**
   - **Does Data table already exist? false → Create a data table**

### F. Setup branch: fetch Slack users and collect mapping responses
34. Add an **HTTP Request** node named **Get TMetric Users for user mapping**.
   - Same config as **Get TMetric Users**

35. Add a **Slack** node named **Get many users**.
   - Resource: user
   - Operation: get all
   - Return all: true
   - Slack OAuth2 auth

36. Add a **Filter** node named **Filter out users**.
   - Conditions:
     - `is_bot` is false
     - `id` is not `USLACKBOT`
     - `deleted` is false

37. Add a **Split In Batches** node named **Send form for each user**.

38. Add a **Slack** node named **Send message and wait for response**.
   - Operation: send and wait
   - Select: channel
   - Choose target channel ID
   - Authentication: Slack OAuth2
   - Message text should mention the current user and ask them to choose a TMetric account
   - Define form as JSON
   - Use custom form with one dropdown field called `TMetric account`
   - Dropdown options expression should come from TMetric user list:
     `$("Get TMetric Users for user mapping").item.json.users.map(...)`

39. Add a **Data Table** node named **Upsert row(s)**.
   - Operation: upsert
   - Table name from **Globals**
   - Filter by `slack_user_id`
   - Store:
     - `slack_user_id`
     - `tmetric_user_id` extracted from selected dropdown option
     - `slack_display_name`

40. Connect:
   - **Does Data table already exist? true → Get TMetric Users for user mapping**
   - **Create a data table → Get TMetric Users for user mapping**
   - **Get TMetric Users for user mapping → Get many users → Filter out users → Send form for each user**
   - **Send form for each user loop output → Send message and wait for response → Upsert row(s) → Send form for each user**

## Required post-build validation
41. Run **Setup** once and verify:
   - Data Table exists
   - Slack form is delivered
   - submitting a selection writes a row

42. Check the Data Table contains:
   - correct `tmetric_user_id`
   - correct `slack_user_id`

43. Run the reporting path manually or temporarily trigger the schedule and verify:
   - TMetric responses are valid
   - users merge with mapping rows
   - Slack DM is sent

## Recommended fixes before production
44. Add handling for empty form submissions in **Upsert row(s)**.
45. Fix the missing node reference in **Calculate data for personalized message**.
46. Fix time-off logic from `includes` to `some`.
47. Consider filtering only approved time-off requests before calculations.
48. Consider replacing positional merge with key-based merge where possible.
49. Consider adding error handling for users missing Slack mappings.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| TMetric API documentation is referenced in the workflow notes. | https://app.tmetric.com/api-docs/ |
| Slack Block Kit Builder is referenced for composing formatted Slack messages. | https://app.slack.com/block-kit-builder/ |
| The workflow author note says the TMetric account ID can be found in the TMetric URL. Example pattern: `https://app.tmetric.com/#/tracker/000000/` | TMetric UI |
| The workflow notes suggest customizing the Slack message in the **Send a message** node. | Internal workflow customization |
| Help/contact note in sticky note: `developers@sailingbyte.com` and `sailingbyte.com` | Project credit / support |
| Important operational note from sticky note: run **Setup** before enabling the weekly schedule. | Workflow operation |
| The sticky notes mention that rerunning setup is safe for new users or corrected mappings. | Workflow maintenance |
| Important limitation: the current logic contains a reference to a missing node named `Percentages of budget spent`, which must be added or removed for the code node to work correctly. | Internal implementation issue |
| Important limitation: empty Slack form submission can break the regex in **Upsert row(s)** unless guarded. | Internal implementation issue |

