Send SaaS trial follow-up emails with MongoDB and Gmail

https://n8nworkflows.xyz/workflows/send-saas-trial-follow-up-emails-with-mongodb-and-gmail-14356


# Send SaaS trial follow-up emails with MongoDB and Gmail

## 1. Workflow Overview

This workflow sends automated follow-up emails to SaaS trial users at specific milestones during their trial period. It runs on a schedule, retrieves trial users from MongoDB, calculates where each user is in the trial lifecycle, and sends one of four Gmail messages depending on the stage:

- Day 3
- Day 7
- Day 13
- Last day before expiry

Typical use cases:
- Trial activation campaigns
- Product adoption nudges
- Upgrade/conversion reminders
- Lifecycle email automation based on subscription dates

### 1.1 Scheduled Input Reception
The workflow starts automatically on a recurring schedule. This acts as the daily entry point for the automation.

### 1.2 Trial User Retrieval and Stage Calculation
It queries MongoDB for users on the `trial` plan, then uses JavaScript logic to compare today’s date against each user’s `plan_start_date` and `plan_end_date`. Only users matching one of the target milestones are kept.

### 1.3 Per-User Iteration and Routing
Eligible users are processed one at a time. A Switch node routes each user to the correct email branch based on the computed `trigger_type`.

### 1.4 Stage-Based Email Delivery
Each branch sends a different Gmail email template tailored to the user’s trial stage.

### 1.5 Branch Rejoin and Loop Continuation
After each email is sent, all branches are merged so the loop can continue processing the next user.

---

## 2. Block-by-Block Analysis

## 2.1 Scheduled Input Reception

**Overview:**  
This block launches the workflow on a recurring schedule. It is the only entry point in the workflow and is intended to run once per day.

**Nodes Involved:**  
- Schedule Trigger

### Node Details

#### Schedule Trigger
- **Type and technical role:** `n8n-nodes-base.scheduleTrigger`  
  Trigger node that starts the workflow automatically on a defined interval.
- **Configuration choices:**  
  The node uses an interval-based schedule. The sticky note indicates the intended behavior is daily execution at midnight, although the raw node configuration shown is a generic interval object and should be verified manually after import.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Find documents`
- **Version-specific requirements:**  
  Uses `typeVersion` 1.3.
- **Edge cases or potential failure types:**  
  - Schedule may not actually be midnight unless explicitly configured in the UI
  - Workflow must be activated in n8n to run automatically
  - Server timezone affects execution time
- **Sub-workflow reference:**  
  None

---

## 2.2 Trial User Retrieval and Stage Calculation

**Overview:**  
This block fetches all trial users from MongoDB and computes whether each user should receive an email today. It normalizes dates to midnight to avoid partial-day calculation issues and assigns a `trigger_type` only when a user is exactly on a target milestone.

**Nodes Involved:**  
- Find documents
- Code in JavaScript

### Node Details

#### Find documents
- **Type and technical role:** `n8n-nodes-base.mongoDb`  
  Queries MongoDB for user records.
- **Configuration choices:**  
  - Collection: `users`
  - Query: users whose `plan` equals `"trial"`
  - No additional query options are configured
- **Key expressions or variables used:**  
  Query:
  - `{"plan": "trial"}`
- **Input and output connections:**  
  - Input: `Schedule Trigger`
  - Output: `Code in JavaScript`
- **Version-specific requirements:**  
  Uses `typeVersion` 1.2.
- **Edge cases or potential failure types:**  
  - MongoDB credential or network failure
  - Wrong database/collection selection
  - Documents missing expected fields such as `plan_start_date`, `plan_end_date`, `email`, or `name`
  - Date fields stored in incompatible formats
- **Sub-workflow reference:**  
  None

#### Code in JavaScript
- **Type and technical role:** `n8n-nodes-base.code`  
  Processes each MongoDB result and determines whether a follow-up email should be sent today.
- **Configuration choices:**  
  The script:
  - Creates `today`
  - Normalizes `today`, `plan_start_date`, and `plan_end_date` to midnight
  - Computes:
    - `diffFromStart` = days elapsed since trial start
    - `diffToEnd` = days remaining until trial end
  - Assigns:
    - `day_3` if `diffFromStart === 3`
    - `day_7` if `diffFromStart === 7`
    - `day_13` if `diffFromStart === 13`
    - `last_day` if `diffToEnd === 1`
  - Returns only matching users
- **Key expressions or variables used:**  
  - `item.json.plan_start_date`
  - `item.json.plan_end_date`
  - `trigger_type`
  - `diffFromStart`
  - `diffToEnd`
- **Input and output connections:**  
  - Input: `Find documents`
  - Output: `Loop Over Items`
- **Version-specific requirements:**  
  Uses `typeVersion` 2.
- **Edge cases or potential failure types:**  
  - Invalid or missing date strings produce `Invalid Date`
  - Timezone differences can shift users into the wrong day bucket
  - If `plan_end_date` and milestone logic overlap, only the first matching condition in the script applies
  - If a trial duration differs from assumptions, some milestones may never match
  - Users not matching exact day counts are intentionally discarded
- **Sub-workflow reference:**  
  None

---

## 2.3 Per-User Iteration and Routing

**Overview:**  
This block processes filtered users individually and routes each one to the proper email path based on `trigger_type`.

**Nodes Involved:**  
- Loop Over Items
- Switch

### Node Details

#### Loop Over Items
- **Type and technical role:** `n8n-nodes-base.splitInBatches`  
  Iterates through the filtered list one item at a time.
- **Configuration choices:**  
  Uses default settings with no custom batch size options shown. In this pattern, the node works as a loop controller.
- **Key expressions or variables used:**  
  None directly.
- **Input and output connections:**  
  - Input: `Code in JavaScript`
  - Output 0: none
  - Output 1: `Switch`
  - Receives return flow from `Merge`
- **Version-specific requirements:**  
  Uses `typeVersion` 3.
- **Edge cases or potential failure types:**  
  - If no users are returned from the Code node, no iteration occurs
  - Miswiring the loop-back connection can cause incomplete processing or stalled loops
- **Sub-workflow reference:**  
  None

#### Switch
- **Type and technical role:** `n8n-nodes-base.switch`  
  Routes each item into one of four paths according to `trigger_type`.
- **Configuration choices:**  
  Four string-equality rules:
  - `day_3`
  - `day_7`
  - `day_13`
  - `last_day`
- **Key expressions or variables used:**  
  - `={{ $json.trigger_type }}`
- **Input and output connections:**  
  - Input: `Loop Over Items`
  - Outputs:
    - Output 0 → `Send day 3 email`
    - Output 1 → `Send day 7 mail`
    - Output 2 → `Send day 13 mail`
    - Output 3 → `Send Last day email`
- **Version-specific requirements:**  
  Uses `typeVersion` 3.4 with rule condition format version 3.
- **Edge cases or potential failure types:**  
  - If `trigger_type` is absent or unexpected, the item will not match any branch
  - Strict type validation is enabled; mismatched types can prevent routing
- **Sub-workflow reference:**  
  None

---

## 2.4 Stage-Based Email Delivery

**Overview:**  
This block sends the actual follow-up emails through Gmail. Each node contains a dedicated subject line and HTML body tailored to the relevant trial stage.

**Nodes Involved:**  
- Send day 3 email
- Send day 7 mail
- Send day 13 mail
- Send Last day email

### Node Details

#### Send day 3 email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the Day 3 onboarding-style email.
- **Configuration choices:**  
  - Recipient: `{{$json.email}}`
  - Subject: `Start hiring smarter with RecruitEase`
  - HTML email body personalized with `{{$json.name}}`
  - Includes CTA link using `{{app_link}}`
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
  - `{{$json.name}}`
  - `{{app_link}}`
- **Input and output connections:**  
  - Input: `Switch` output 0
  - Output: `Merge` input 0
- **Version-specific requirements:**  
  Uses `typeVersion` 2.2.
- **Edge cases or potential failure types:**  
  - Gmail OAuth credential issues
  - Invalid recipient email
  - `name` missing, causing awkward greeting
  - `app_link` is referenced as a template placeholder but is not defined anywhere in the workflow JSON; it must be hardcoded, injected upstream, or replaced with a valid expression
- **Sub-workflow reference:**  
  None

#### Send day 7 mail
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the Day 7 midpoint engagement email.
- **Configuration choices:**  
  - Recipient: `{{ $json.email }}`
  - Subject: `You're halfway there — unlock smarter hiring`
  - HTML content personalized with `{{$json.name}}`
  - Includes CTA link using `{{app_link}}`
- **Key expressions or variables used:**  
  - `{{ $json.email }}`
  - `{{$json.name}}`
  - `{{app_link}}`
- **Input and output connections:**  
  - Input: `Switch` output 1
  - Output: `Merge` input 1
- **Version-specific requirements:**  
  Uses `typeVersion` 2.2.
- **Edge cases or potential failure types:**  
  - Same Gmail and placeholder risks as above
  - The recipient field lacks the leading `=` used elsewhere; depending on n8n expression parsing in the field, this may still work if configured as an expression, but it should be verified after import
- **Sub-workflow reference:**  
  None

#### Send day 13 mail
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the near-expiry reminder.
- **Configuration choices:**  
  - Recipient: `{{$json.email}}`
  - Subject: `Your RecruitEase trial is almost over`
  - HTML body personalized with `{{$json.name}}`
  - Includes CTA link using `{{app_link}}`
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
  - `{{$json.name}}`
  - `{{app_link}}`
- **Input and output connections:**  
  - Input: `Switch` output 2
  - Output: `Merge` input 2
- **Version-specific requirements:**  
  Uses `typeVersion` 2.2.
- **Edge cases or potential failure types:**  
  - Same Gmail and undefined placeholder risks as above
- **Sub-workflow reference:**  
  None

#### Send Last day email
- **Type and technical role:** `n8n-nodes-base.gmail`  
  Sends the final-day conversion email.
- **Configuration choices:**  
  - Recipient: `{{$json.email}}`
  - Subject: `Last day — don’t lose your hiring progress`
  - HTML body personalized with `{{$json.name}}`
  - Includes CTA link using `{{upgrade_link}}`
- **Key expressions or variables used:**  
  - `={{ $json.email }}`
  - `{{$json.name}}`
  - `{{upgrade_link}}`
- **Input and output connections:**  
  - Input: `Switch` output 3
  - Output: `Merge` input 3
- **Version-specific requirements:**  
  Uses `typeVersion` 2.2.
- **Edge cases or potential failure types:**  
  - Same Gmail risks
  - `upgrade_link` is not defined anywhere in the workflow JSON and must be provided another way
- **Sub-workflow reference:**  
  None

---

## 2.5 Branch Rejoin and Loop Continuation

**Overview:**  
This block reunifies the four mutually exclusive email branches and feeds control back into the loop so the next eligible user can be processed.

**Nodes Involved:**  
- Merge

### Node Details

#### Merge
- **Type and technical role:** `n8n-nodes-base.merge`  
  Consolidates outputs from the four Gmail nodes into a single stream.
- **Configuration choices:**  
  - `numberInputs: 4`
  - Used as a branch joiner before returning to the loop controller
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Inputs:
    - `Send day 3 email`
    - `Send day 7 mail`
    - `Send day 13 mail`
    - `Send Last day email`
  - Output: `Loop Over Items`
- **Version-specific requirements:**  
  Uses `typeVersion` 3.2.
- **Edge cases or potential failure types:**  
  - If branch wiring is changed incorrectly, loop continuation may break
  - Merge behavior should be tested after edits because it is used in a loop-back pattern rather than simple data aggregation
- **Sub-workflow reference:**  
  None

---

## 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Schedule Trigger | Schedule Trigger | Starts the workflow on a recurring schedule |  | Find documents | # Automated SaaS Trial Expiry Nudge Sequence<br>This workflow automates follow-up emails for users during their SaaS trial period. It ensures timely engagement by sending personalized nudges at key intervals—Day 3, Day 7, Day 13, and the final day before trial expiry. This helps improve user activation, retention, and conversion without manual tracking.<br>The workflow runs daily, checks user subscription data, and determines which stage each user is in. Based on this, it triggers the appropriate email communication. Each email is designed to guide users toward product adoption and encourage conversion before the trial ends.<br>### How it works<br>- Workflow runs daily at midnight<br>- User data is fetched from database/spreadsheet<br>- Trial dates are analyzed to determine stage<br>- Users are tagged (Day 3, 7, 13, Last Day)<br>- Loop processes each eligible user<br>- Relevant email is sent based on stage<br>### Setup<br>- Configure Schedule Trigger (daily midnight).<br>- Connect your database or spreadsheet.<br>- Ensure plan start/end dates are available.<br>- Connect Gmail account credentials (or any email provider).<br>- Emails are based on a demo SaaS — update content to match your product.<br>## Step 1 : Fetch & Filter Trial Users<br>Runs daily at midnight, retrieves user and plan data, and filters users based on their trial stage (Day 3, 7, 13, or last day before expiry). |
| Find documents | MongoDB | Retrieves users with plan `trial` from MongoDB | Schedule Trigger | Code in JavaScript | # Automated SaaS Trial Expiry Nudge Sequence<br>This workflow automates follow-up emails for users during their SaaS trial period. It ensures timely engagement by sending personalized nudges at key intervals—Day 3, Day 7, Day 13, and the final day before trial expiry. This helps improve user activation, retention, and conversion without manual tracking.<br>The workflow runs daily, checks user subscription data, and determines which stage each user is in. Based on this, it triggers the appropriate email communication. Each email is designed to guide users toward product adoption and encourage conversion before the trial ends.<br>### How it works<br>- Workflow runs daily at midnight<br>- User data is fetched from database/spreadsheet<br>- Trial dates are analyzed to determine stage<br>- Users are tagged (Day 3, 7, 13, Last Day)<br>- Loop processes each eligible user<br>- Relevant email is sent based on stage<br>### Setup<br>- Configure Schedule Trigger (daily midnight).<br>- Connect your database or spreadsheet.<br>- Ensure plan start/end dates are available.<br>- Connect Gmail account credentials (or any email provider).<br>- Emails are based on a demo SaaS — update content to match your product.<br>## Step 1 : Fetch & Filter Trial Users<br>Runs daily at midnight, retrieves user and plan data, and filters users based on their trial stage (Day 3, 7, 13, or last day before expiry). |
| Code in JavaScript | Code | Calculates trial milestone and filters eligible users | Find documents | Loop Over Items | # Automated SaaS Trial Expiry Nudge Sequence<br>This workflow automates follow-up emails for users during their SaaS trial period. It ensures timely engagement by sending personalized nudges at key intervals—Day 3, Day 7, Day 13, and the final day before trial expiry. This helps improve user activation, retention, and conversion without manual tracking.<br>The workflow runs daily, checks user subscription data, and determines which stage each user is in. Based on this, it triggers the appropriate email communication. Each email is designed to guide users toward product adoption and encourage conversion before the trial ends.<br>### How it works<br>- Workflow runs daily at midnight<br>- User data is fetched from database/spreadsheet<br>- Trial dates are analyzed to determine stage<br>- Users are tagged (Day 3, 7, 13, Last Day)<br>- Loop processes each eligible user<br>- Relevant email is sent based on stage<br>### Setup<br>- Configure Schedule Trigger (daily midnight).<br>- Connect your database or spreadsheet.<br>- Ensure plan start/end dates are available.<br>- Connect Gmail account credentials (or any email provider).<br>- Emails are based on a demo SaaS — update content to match your product.<br>## Step 1 : Fetch & Filter Trial Users<br>Runs daily at midnight, retrieves user and plan data, and filters users based on their trial stage (Day 3, 7, 13, or last day before expiry). |
| Loop Over Items | Split In Batches | Iterates through eligible users one by one | Code in JavaScript, Merge | Switch | # Automated SaaS Trial Expiry Nudge Sequence<br>This workflow automates follow-up emails for users during their SaaS trial period. It ensures timely engagement by sending personalized nudges at key intervals—Day 3, Day 7, Day 13, and the final day before trial expiry. This helps improve user activation, retention, and conversion without manual tracking.<br>The workflow runs daily, checks user subscription data, and determines which stage each user is in. Based on this, it triggers the appropriate email communication. Each email is designed to guide users toward product adoption and encourage conversion before the trial ends.<br>### How it works<br>- Workflow runs daily at midnight<br>- User data is fetched from database/spreadsheet<br>- Trial dates are analyzed to determine stage<br>- Users are tagged (Day 3, 7, 13, Last Day)<br>- Loop processes each eligible user<br>- Relevant email is sent based on stage<br>### Setup<br>- Configure Schedule Trigger (daily midnight).<br>- Connect your database or spreadsheet.<br>- Ensure plan start/end dates are available.<br>- Connect Gmail account credentials (or any email provider).<br>- Emails are based on a demo SaaS — update content to match your product.<br>## Step 2 : Send Stage-Based Nudge Emails<br>Loops through filtered users and sends the appropriate email based on trial stage. |
| Switch | Switch | Routes each user to the correct email branch | Loop Over Items | Send day 3 email, Send day 7 mail, Send day 13 mail, Send Last day email | # Automated SaaS Trial Expiry Nudge Sequence<br>This workflow automates follow-up emails for users during their SaaS trial period. It ensures timely engagement by sending personalized nudges at key intervals—Day 3, Day 7, Day 13, and the final day before trial expiry. This helps improve user activation, retention, and conversion without manual tracking.<br>The workflow runs daily, checks user subscription data, and determines which stage each user is in. Based on this, it triggers the appropriate email communication. Each email is designed to guide users toward product adoption and encourage conversion before the trial ends.<br>### How it works<br>- Workflow runs daily at midnight<br>- User data is fetched from database/spreadsheet<br>- Trial dates are analyzed to determine stage<br>- Users are tagged (Day 3, 7, 13, Last Day)<br>- Loop processes each eligible user<br>- Relevant email is sent based on stage<br>### Setup<br>- Configure Schedule Trigger (daily midnight).<br>- Connect your database or spreadsheet.<br>- Ensure plan start/end dates are available.<br>- Connect Gmail account credentials (or any email provider).<br>- Emails are based on a demo SaaS — update content to match your product.<br>## Step 2 : Send Stage-Based Nudge Emails<br>Loops through filtered users and sends the appropriate email based on trial stage. |
| Send day 3 email | Gmail | Sends Day 3 email | Switch | Merge | # Automated SaaS Trial Expiry Nudge Sequence<br>This workflow automates follow-up emails for users during their SaaS trial period. It ensures timely engagement by sending personalized nudges at key intervals—Day 3, Day 7, Day 13, and the final day before trial expiry. This helps improve user activation, retention, and conversion without manual tracking.<br>The workflow runs daily, checks user subscription data, and determines which stage each user is in. Based on this, it triggers the appropriate email communication. Each email is designed to guide users toward product adoption and encourage conversion before the trial ends.<br>### How it works<br>- Workflow runs daily at midnight<br>- User data is fetched from database/spreadsheet<br>- Trial dates are analyzed to determine stage<br>- Users are tagged (Day 3, 7, 13, Last Day)<br>- Loop processes each eligible user<br>- Relevant email is sent based on stage<br>### Setup<br>- Configure Schedule Trigger (daily midnight).<br>- Connect your database or spreadsheet.<br>- Ensure plan start/end dates are available.<br>- Connect Gmail account credentials (or any email provider).<br>- Emails are based on a demo SaaS — update content to match your product.<br>## Step 2 : Send Stage-Based Nudge Emails<br>Loops through filtered users and sends the appropriate email based on trial stage. |
| Send day 7 mail | Gmail | Sends Day 7 email | Switch | Merge | # Automated SaaS Trial Expiry Nudge Sequence<br>This workflow automates follow-up emails for users during their SaaS trial period. It ensures timely engagement by sending personalized nudges at key intervals—Day 3, Day 7, Day 13, and the final day before trial expiry. This helps improve user activation, retention, and conversion without manual tracking.<br>The workflow runs daily, checks user subscription data, and determines which stage each user is in. Based on this, it triggers the appropriate email communication. Each email is designed to guide users toward product adoption and encourage conversion before the trial ends.<br>### How it works<br>- Workflow runs daily at midnight<br>- User data is fetched from database/spreadsheet<br>- Trial dates are analyzed to determine stage<br>- Users are tagged (Day 3, 7, 13, Last Day)<br>- Loop processes each eligible user<br>- Relevant email is sent based on stage<br>### Setup<br>- Configure Schedule Trigger (daily midnight).<br>- Connect your database or spreadsheet.<br>- Ensure plan start/end dates are available.<br>- Connect Gmail account credentials (or any email provider).<br>- Emails are based on a demo SaaS — update content to match your product.<br>## Step 2 : Send Stage-Based Nudge Emails<br>Loops through filtered users and sends the appropriate email based on trial stage. |
| Send day 13 mail | Gmail | Sends Day 13 email | Switch | Merge | # Automated SaaS Trial Expiry Nudge Sequence<br>This workflow automates follow-up emails for users during their SaaS trial period. It ensures timely engagement by sending personalized nudges at key intervals—Day 3, Day 7, Day 13, and the final day before trial expiry. This helps improve user activation, retention, and conversion without manual tracking.<br>The workflow runs daily, checks user subscription data, and determines which stage each user is in. Based on this, it triggers the appropriate email communication. Each email is designed to guide users toward product adoption and encourage conversion before the trial ends.<br>### How it works<br>- Workflow runs daily at midnight<br>- User data is fetched from database/spreadsheet<br>- Trial dates are analyzed to determine stage<br>- Users are tagged (Day 3, 7, 13, Last Day)<br>- Loop processes each eligible user<br>- Relevant email is sent based on stage<br>### Setup<br>- Configure Schedule Trigger (daily midnight).<br>- Connect your database or spreadsheet.<br>- Ensure plan start/end dates are available.<br>- Connect Gmail account credentials (or any email provider).<br>- Emails are based on a demo SaaS — update content to match your product.<br>## Step 2 : Send Stage-Based Nudge Emails<br>Loops through filtered users and sends the appropriate email based on trial stage. |
| Send Last day email | Gmail | Sends final-day email | Switch | Merge | # Automated SaaS Trial Expiry Nudge Sequence<br>This workflow automates follow-up emails for users during their SaaS trial period. It ensures timely engagement by sending personalized nudges at key intervals—Day 3, Day 7, Day 13, and the final day before trial expiry. This helps improve user activation, retention, and conversion without manual tracking.<br>The workflow runs daily, checks user subscription data, and determines which stage each user is in. Based on this, it triggers the appropriate email communication. Each email is designed to guide users toward product adoption and encourage conversion before the trial ends.<br>### How it works<br>- Workflow runs daily at midnight<br>- User data is fetched from database/spreadsheet<br>- Trial dates are analyzed to determine stage<br>- Users are tagged (Day 3, 7, 13, Last Day)<br>- Loop processes each eligible user<br>- Relevant email is sent based on stage<br>### Setup<br>- Configure Schedule Trigger (daily midnight).<br>- Connect your database or spreadsheet.<br>- Ensure plan start/end dates are available.<br>- Connect Gmail account credentials (or any email provider).<br>- Emails are based on a demo SaaS — update content to match your product.<br>## Step 2 : Send Stage-Based Nudge Emails<br>Loops through filtered users and sends the appropriate email based on trial stage. |
| Merge | Merge | Rejoins branches and continues loop | Send day 3 email, Send day 7 mail, Send day 13 mail, Send Last day email | Loop Over Items | # Automated SaaS Trial Expiry Nudge Sequence<br>This workflow automates follow-up emails for users during their SaaS trial period. It ensures timely engagement by sending personalized nudges at key intervals—Day 3, Day 7, Day 13, and the final day before trial expiry. This helps improve user activation, retention, and conversion without manual tracking.<br>The workflow runs daily, checks user subscription data, and determines which stage each user is in. Based on this, it triggers the appropriate email communication. Each email is designed to guide users toward product adoption and encourage conversion before the trial ends.<br>### How it works<br>- Workflow runs daily at midnight<br>- User data is fetched from database/spreadsheet<br>- Trial dates are analyzed to determine stage<br>- Users are tagged (Day 3, 7, 13, Last Day)<br>- Loop processes each eligible user<br>- Relevant email is sent based on stage<br>### Setup<br>- Configure Schedule Trigger (daily midnight).<br>- Connect your database or spreadsheet.<br>- Ensure plan start/end dates are available.<br>- Connect Gmail account credentials (or any email provider).<br>- Emails are based on a demo SaaS — update content to match your product.<br>## Step 2 : Send Stage-Based Nudge Emails<br>Loops through filtered users and sends the appropriate email based on trial stage. |
| Sticky Note6 | Sticky Note | Documentation note on overall workflow |  |  | # Automated SaaS Trial Expiry Nudge Sequence<br>This workflow automates follow-up emails for users during their SaaS trial period. It ensures timely engagement by sending personalized nudges at key intervals—Day 3, Day 7, Day 13, and the final day before trial expiry. This helps improve user activation, retention, and conversion without manual tracking.<br>The workflow runs daily, checks user subscription data, and determines which stage each user is in. Based on this, it triggers the appropriate email communication. Each email is designed to guide users toward product adoption and encourage conversion before the trial ends.<br>### How it works<br>- Workflow runs daily at midnight<br>- User data is fetched from database/spreadsheet<br>- Trial dates are analyzed to determine stage<br>- Users are tagged (Day 3, 7, 13, Last Day)<br>- Loop processes each eligible user<br>- Relevant email is sent based on stage<br>### Setup<br>- Configure Schedule Trigger (daily midnight).<br>- Connect your database or spreadsheet.<br>- Ensure plan start/end dates are available.<br>- Connect Gmail account credentials (or any email provider).<br>- Emails are based on a demo SaaS — update content to match your product. |
| Sticky Note7 | Sticky Note | Documentation note for email sending block |  |  | ## Step 2 : Send Stage-Based Nudge Emails<br>Loops through filtered users and sends the appropriate email based on trial stage. |
| Sticky Note8 | Sticky Note | Documentation note for fetch/filter block |  |  | ## Step 1 : Fetch & Filter Trial Users<br>Runs daily at midnight, retrieves user and plan data, and filters users based on their trial stage (Day 3, 7, 13, or last day before expiry). |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**  
   Name it something like: `Send SaaS trial follow-up emails with MongoDB and Gmail`.

2. **Add a Schedule Trigger node**
   - Node type: **Schedule Trigger**
   - Configure it to run **daily**
   - Set the preferred execution time to **midnight**
   - Verify the workflow timezone in n8n settings so the day calculations align with your users or your business timezone

3. **Add a MongoDB node**
   - Node type: **MongoDB**
   - Name it: `Find documents`
   - Operation: find/query documents
   - Credentials: configure MongoDB credentials
     - Host / connection string
     - Database name
     - Authentication if required
   - Collection: `users`
   - Query:
     ```json
     {
       "plan": "trial"
     }
     ```
   - Connect `Schedule Trigger -> Find documents`

4. **Add a Code node**
   - Node type: **Code**
   - Name it: `Code in JavaScript`
   - Language: JavaScript
   - Paste logic that:
     - normalizes current date to midnight
     - normalizes each user’s `plan_start_date` and `plan_end_date`
     - calculates days since start and days until end
     - sets `trigger_type` to:
       - `day_3`
       - `day_7`
       - `day_13`
       - `last_day`
     - returns only matching users
   - Use this logic:
     ```javascript
     const today = new Date();
     today.setHours(0, 0, 0, 0);

     return items.map(item => {
       const start = new Date(item.json.plan_start_date);
       const end = new Date(item.json.plan_end_date);

       start.setHours(0, 0, 0, 0);
       end.setHours(0, 0, 0, 0);

       const diffFromStart = Math.floor((today - start) / (1000 * 60 * 60 * 24));
       const diffToEnd = Math.floor((end - today) / (1000 * 60 * 60 * 24));

       let trigger_type = null;

       if (diffFromStart === 3) trigger_type = "day_3";
       else if (diffFromStart === 7) trigger_type = "day_7";
       else if (diffFromStart === 13) trigger_type = "day_13";
       else if (diffToEnd === 1) trigger_type = "last_day";

       if (trigger_type) {
         return {
           json: {
             ...item.json,
             trigger_type,
           }
         };
       }
     }).filter(Boolean);
     ```
   - Connect `Find documents -> Code in JavaScript`

5. **Add a Split In Batches node**
   - Node type: **Loop Over Items / Split In Batches**
   - Name it: `Loop Over Items`
   - Keep default settings unless you want larger batch processing
   - Connect `Code in JavaScript -> Loop Over Items`

6. **Add a Switch node**
   - Node type: **Switch**
   - Name it: `Switch`
   - Configure four rules based on `{{$json.trigger_type}}`
   - Rule 1: equals `day_3`
   - Rule 2: equals `day_7`
   - Rule 3: equals `day_13`
   - Rule 4: equals `last_day`
   - Prefer strict string comparison
   - Connect `Loop Over Items -> Switch` using the loop output that emits current items

7. **Add the Day 3 Gmail node**
   - Node type: **Gmail**
   - Name it: `Send day 3 email`
   - Operation: send email
   - Credentials: configure Gmail OAuth2
   - To: `{{$json.email}}`
   - Subject: `Start hiring smarter with RecruitEase`
   - Message type: HTML
   - Use the Day 3 HTML content
   - Replace `{{app_link}}` with one of the following:
     - a hardcoded URL, or
     - an n8n expression such as `{{$json.app_link}}`, if supplied in data
   - Connect `Switch output 0 -> Send day 3 email`

8. **Add the Day 7 Gmail node**
   - Node type: **Gmail**
   - Name it: `Send day 7 mail`
   - To: `{{$json.email}}`
   - Subject: `You're halfway there — unlock smarter hiring`
   - Message type: HTML
   - Use the Day 7 HTML content
   - Replace `{{app_link}}` with a real URL or valid expression
   - Connect `Switch output 1 -> Send day 7 mail`

9. **Add the Day 13 Gmail node**
   - Node type: **Gmail**
   - Name it: `Send day 13 mail`
   - To: `{{$json.email}}`
   - Subject: `Your RecruitEase trial is almost over`
   - Message type: HTML
   - Use the Day 13 HTML content
   - Replace `{{app_link}}` with a real URL or valid expression
   - Connect `Switch output 2 -> Send day 13 mail`

10. **Add the Last Day Gmail node**
    - Node type: **Gmail**
    - Name it: `Send Last day email`
    - To: `{{$json.email}}`
    - Subject: `Last day — don’t lose your hiring progress`
    - Message type: HTML
    - Use the final-day HTML content
    - Replace `{{upgrade_link}}` with a real upgrade URL or valid expression such as `{{$json.upgrade_link}}`
    - Connect `Switch output 3 -> Send Last day email`

11. **Add a Merge node**
    - Node type: **Merge**
    - Name it: `Merge`
    - Set number of inputs to **4**
    - Connect:
      - `Send day 3 email -> Merge input 0`
      - `Send day 7 mail -> Merge input 1`
      - `Send day 13 mail -> Merge input 2`
      - `Send Last day email -> Merge input 3`

12. **Close the loop**
    - Connect `Merge -> Loop Over Items`
    - This allows the loop controller to continue with the next eligible user after one email is sent

13. **Add optional sticky notes for clarity**
    - Add one note describing the overall workflow purpose
    - Add one note for the fetch/filter stage
    - Add one note for the email sending stage

14. **Prepare required user data structure in MongoDB**
    Each `users` document should contain at least:
    - `plan`
    - `plan_start_date`
    - `plan_end_date`
    - `email`
    - `name`

    Recommended additional fields if you want dynamic links:
    - `app_link`
    - `upgrade_link`

15. **Configure credentials**
    - **MongoDB credentials**
      - Ensure the node points to the correct database containing the `users` collection
    - **Gmail credentials**
      - Use OAuth2
      - Ensure the connected Gmail account is authorized to send messages
      - Check Gmail sending limits if sending at scale

16. **Test with sample documents**
    Insert test users with trial dates that should trigger each path:
    - one with start date 3 days ago
    - one with start date 7 days ago
    - one with start date 13 days ago
    - one whose end date is tomorrow

17. **Validate expressions inside email bodies**
    - Confirm `{{$json.name}}` renders correctly
    - Replace placeholder variables like `{{app_link}}` and `{{upgrade_link}}` with valid expressions or fixed URLs
    - If left unchanged, these links may not render as intended

18. **Activate the workflow**
    - Save the workflow
    - Activate it so the Schedule Trigger runs automatically

### Credential and implementation notes
- **MongoDB:** make sure date fields are stored consistently, ideally ISO date strings or BSON dates.
- **Gmail:** if using a Google Workspace account, confirm org policies permit API-based sending.
- **Timezone:** this workflow depends heavily on exact day boundaries. Align:
  - n8n instance timezone
  - MongoDB date storage timezone
  - business logic timezone

### Sub-workflow setup
This workflow does **not** use any sub-workflow node and has **one entry point only**: `Schedule Trigger`.

---

## 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| Automated SaaS Trial Expiry Nudge Sequence: This workflow automates follow-up emails for users during their SaaS trial period. It ensures timely engagement by sending personalized nudges at key intervals—Day 3, Day 7, Day 13, and the final day before trial expiry. This helps improve user activation, retention, and conversion without manual tracking. | Overall workflow purpose |
| Workflow runs daily, checks user subscription data, and determines which stage each user is in. Based on this, it triggers the appropriate email communication. Each email is designed to guide users toward product adoption and encourage conversion before the trial ends. | Overall workflow behavior |
| How it works: Workflow runs daily at midnight; user data is fetched from database/spreadsheet; trial dates are analyzed to determine stage; users are tagged (Day 3, 7, 13, Last Day); loop processes each eligible user; relevant email is sent based on stage. | Functional summary |
| Setup: Configure Schedule Trigger (daily midnight); connect your database or spreadsheet; ensure plan start/end dates are available; connect Gmail account credentials (or any email provider); emails are based on a demo SaaS — update content to match your product. | Implementation guidance |
| Step 1 : Fetch & Filter Trial Users | Fetch/filter block |
| Runs daily at midnight, retrieves user and plan data, and filters users based on their trial stage (Day 3, 7, 13, or last day before expiry). | Fetch/filter block |
| Step 2 : Send Stage-Based Nudge Emails | Email delivery block |
| Loops through filtered users and sends the appropriate email based on trial stage. | Email delivery block |