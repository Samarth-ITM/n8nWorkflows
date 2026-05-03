Weekly Business & Home Task Planner with Notion, Email Digests and Telegram

https://n8nworkflows.xyz/workflows/weekly-business---home-task-planner-with-notion--email-digests-and-telegram-7789


# Weekly Business & Home Task Planner with Notion, Email Digests and Telegram

### 1. Workflow Overview

This workflow, titled **"Household & Business Weekly Rhythm Planner (v1)"**, automates the weekly planning process by gathering tasks from Notion (if enabled), combining them with sample or user-defined tasks, and then distributing the weekly plan via email and Telegram notifications. It targets individuals or small teams seeking a streamlined way to organize and communicate weekly business and home-related tasks.  

The workflow is logically divided into these blocks:

- **1.1 Trigger & Configuration Setup:** Receives a weekly or manual trigger and sets user configuration parameters.
- **1.2 Task Acquisition:** Checks if Notion integration is enabled; if yes, fetches tasks from Notion and extracts them.
- **1.3 Plan Construction:** Builds the weekly plan by combining fetched or sample tasks into a digestible format.
- **1.4 Communication & Notifications:** Sends the weekly plan via email to the user and optionally to a partner, and sends a Telegram notification if enabled.
- **1.5 Error Handling:** Catches runtime errors and notifies the owner by email.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger & Configuration Setup

- **Overview:**  
  This block initializes the workflow execution via scheduled or manual triggers and sets up user-specific configurations necessary for downstream nodes.

- **Nodes Involved:**  
  - Weekly Cron  
  - Manual Trigger  
  - Set: User Config

- **Node Details:**  

  - **Weekly Cron**  
    - Type: Cron Trigger  
    - Role: Automatically triggers the workflow weekly at a configured schedule (default schedule not explicitly set in JSON).  
    - Inputs: None (start node)  
    - Outputs: To `Set: User Config`  
    - Failures: Cron misconfiguration could cause missed triggers.  

  - **Manual Trigger**  
    - Type: Manual Trigger  
    - Role: Allows manual execution for testing or on-demand runs.  
    - Inputs: None (start node)  
    - Outputs: To `Function: Sample Tasks` and `Set: User Config` concurrently.  
    - Failures: None inherent; user action required.  

  - **Set: User Config**  
    - Type: Set Node  
    - Role: Holds user-specific settings such as toggles for Notion, partner email, Telegram, and other config flags (details not specified in JSON).  
    - Inputs: From `Weekly Cron` or `Manual Trigger`  
    - Outputs: To `If: Notion Enabled?`  
    - Failures: Misconfigured variables or missing parameters could cause logic errors downstream.

---

#### 2.2 Task Acquisition

- **Overview:**  
  This block conditionally fetches tasks from Notion if the integration is enabled; otherwise, it relies on sample tasks. It extracts and prepares tasks for the weekly plan.

- **Nodes Involved:**  
  - If: Notion Enabled?  
  - Notion: Get Tasks  
  - Function: Extract Notion Tasks  
  - Function: Sample Tasks

- **Node Details:**  

  - **If: Notion Enabled?**  
    - Type: If Node  
    - Role: Branches workflow depending on whether Notion integration is enabled in user config.  
    - Inputs: From `Set: User Config`  
    - Outputs: True → `Notion: Get Tasks`; False → skips Notion nodes.  
    - Failures: Expression evaluation errors if config variables missing.

  - **Notion: Get Tasks**  
    - Type: Notion Node  
    - Role: Retrieves tasks from Notion database(s) configured in credentials and parameters (specific database IDs or filters not specified).  
    - Inputs: From `If: Notion Enabled?` (True branch)  
    - Outputs: To `Function: Extract Notion Tasks`  
    - Failures: Authentication errors, API rate limits, empty responses, or misconfigured database IDs.

  - **Function: Extract Notion Tasks**  
    - Type: Function Node (JavaScript)  
    - Role: Parses and transforms the raw Notion data into a structured list of tasks suitable for the weekly plan.  
    - Inputs: From `Notion: Get Tasks`  
    - Outputs: To `Function: Build Weekly Plan`  
    - Key Logic: Extract task titles, due dates, priorities, or other metadata from Notion response.  
    - Failures: Parsing errors if Notion schema changes or response is malformed.

  - **Function: Sample Tasks**  
    - Type: Function Node  
    - Role: Provides a predefined set of sample tasks when Notion is disabled or manual trigger is used.  
    - Inputs: From `Manual Trigger`  
    - Outputs: To `Function: Build Weekly Plan`  
    - Failures: Logic errors if sample data is malformed.

---

#### 2.3 Plan Construction

- **Overview:**  
  Combines extracted or sample tasks into a formatted weekly plan digest ready for distribution.

- **Nodes Involved:**  
  - Function: Build Weekly Plan

- **Node Details:**  

  - **Function: Build Weekly Plan**  
    - Type: Function Node  
    - Role: Aggregates task lists, formats them (likely as HTML or plain text), and prepares the content for emails and Telegram messages.  
    - Inputs: From `Function: Extract Notion Tasks` or `Function: Sample Tasks`  
    - Outputs: To `Email: Send to You`, `If: Partner Enabled?`, and `If: Telegram Enabled?`  
    - Failures: Formatting or data structure errors if inputs are inconsistent.

---

#### 2.4 Communication & Notifications

- **Overview:**  
  Sends the weekly plan digest to designated recipients via email and optionally posts a notification on Telegram.

- **Nodes Involved:**  
  - Email: Send to You  
  - If: Partner Enabled?  
  - Email: Send to Partner  
  - If: Telegram Enabled?  
  - Telegram: Notify

- **Node Details:**  

  - **Email: Send to You**  
    - Type: Email Send Node  
    - Role: Sends the weekly plan email to the primary user.  
    - Inputs: From `Function: Build Weekly Plan`  
    - Configuration: Uses configured SMTP or email credentials; email parameters (To, Subject, Body) likely use expressions based on plan content.  
    - Failures: SMTP authentication errors, network issues, invalid email addresses.

  - **If: Partner Enabled?**  
    - Type: If Node  
    - Role: Checks if a partner email recipient is enabled in config.  
    - Inputs: From `Function: Build Weekly Plan`  
    - Outputs: True → `Email: Send to Partner`; False → no action.  
    - Failures: Misconfigured partner email flag or missing email address.

  - **Email: Send to Partner**  
    - Type: Email Send Node  
    - Role: Sends the weekly plan email to the partner recipient.  
    - Inputs: From `If: Partner Enabled?` (True branch)  
    - Failures: Same as primary email node.

  - **If: Telegram Enabled?**  
    - Type: If Node  
    - Role: Checks if Telegram notifications are enabled.  
    - Inputs: From `Function: Build Weekly Plan`  
    - Outputs: True → `Telegram: Notify`; False → no action.  
    - Failures: Misconfigured Telegram flag.

  - **Telegram: Notify**  
    - Type: Telegram Node  
    - Role: Sends a Telegram message containing the weekly plan or notification to a configured chat ID.  
    - Inputs: From `If: Telegram Enabled?` (True branch)  
    - Configuration: Requires Telegram Bot API credentials and target chat ID.  
    - Failures: Bot token invalid, chat ID wrong, network errors.

---

#### 2.5 Error Handling

- **Overview:**  
  This block captures any runtime errors from the workflow and notifies the owner via email.

- **Nodes Involved:**  
  - On Error  
  - On Error → Email Owner

- **Node Details:**  

  - **On Error**  
    - Type: Error Trigger  
    - Role: Listens for errors thrown anywhere in the workflow.  
    - Outputs: To `On Error → Email Owner`  
    - Failures: None (this node is for error capturing).

  - **On Error → Email Owner**  
    - Type: Email Send Node  
    - Role: Sends an error notification email to the workflow owner including error details.  
    - Inputs: From `On Error`  
    - Failures: Same as other email nodes; failure to send error email can obscure error visibility.

---

### 3. Summary Table

| Node Name                | Node Type           | Functional Role                          | Input Node(s)            | Output Node(s)                      | Sticky Note                                                                                             |
|--------------------------|---------------------|----------------------------------------|--------------------------|-----------------------------------|-------------------------------------------------------------------------------------------------------|
| Sticky: README           | Sticky Note         | Documentation placeholder              | -                        | -                                 |                                                                                                       |
| Sticky: Prereqs + CSV    | Sticky Note         | Documentation placeholder              | -                        | -                                 |                                                                                                       |
| Sticky: Setup Checklist  | Sticky Note         | Documentation placeholder              | -                        | -                                 |                                                                                                       |
| Sticky: Testing & Behavior| Sticky Note        | Documentation placeholder              | -                        | -                                 |                                                                                                       |
| Sticky: Compliance + Troubleshooting | Sticky Note| Documentation placeholder              | -                        | -                                 |                                                                                                       |
| Weekly Cron              | Cron Trigger        | Scheduled weekly workflow trigger      | -                        | Set: User Config                   |                                                                                                       |
| Manual Trigger           | Manual Trigger      | Manual workflow trigger                 | -                        | Function: Sample Tasks, Set: User Config |                                                                                                       |
| Set: User Config         | Set Node            | User configuration parameters          | Weekly Cron, Manual Trigger | If: Notion Enabled?               |                                                                                                       |
| If: Notion Enabled?      | If Node             | Branch to fetch Notion tasks if enabled| Set: User Config          | Notion: Get Tasks, (else skip)     |                                                                                                       |
| Notion: Get Tasks        | Notion Node         | Fetch tasks from Notion database        | If: Notion Enabled? (true) | Function: Extract Notion Tasks     |                                                                                                       |
| Function: Extract Notion Tasks | Function Node  | Parse and extract tasks from Notion    | Notion: Get Tasks         | Function: Build Weekly Plan        |                                                                                                       |
| Function: Sample Tasks   | Function Node       | Provide sample tasks when Notion disabled | Manual Trigger          | Function: Build Weekly Plan        |                                                                                                       |
| Function: Build Weekly Plan | Function Node    | Build formatted weekly task plan       | Function: Extract Notion Tasks, Function: Sample Tasks | Email: Send to You, If: Partner Enabled?, If: Telegram Enabled? |                                                                                                       |
| Email: Send to You       | Email Send Node     | Send weekly plan to primary user        | Function: Build Weekly Plan | -                               |                                                                                                       |
| If: Partner Enabled?     | If Node             | Check if partner email enabled          | Function: Build Weekly Plan | Email: Send to Partner, (else skip)|                                                                                                       |
| Email: Send to Partner   | Email Send Node     | Send weekly plan to partner              | If: Partner Enabled? (true) | -                               |                                                                                                       |
| If: Telegram Enabled?    | If Node             | Check if Telegram notification enabled  | Function: Build Weekly Plan | Telegram: Notify, (else skip)     |                                                                                                       |
| Telegram: Notify         | Telegram Node       | Send weekly plan notification via Telegram | If: Telegram Enabled? (true) | -                              |                                                                                                       |
| On Error                 | Error Trigger       | Catch workflow errors                    | -                        | On Error → Email Owner             |                                                                                                       |
| On Error → Email Owner   | Email Send Node     | Send error notification to owner        | On Error                  | -                                 |                                                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Cron** node named `Weekly Cron` with a schedule to trigger weekly (e.g., every Monday 8 AM).  
   - Add a **Manual Trigger** node named `Manual Trigger` for manual runs.

2. **Set User Configuration:**  
   - Add a **Set** node named `Set: User Config`.  
   - Configure key parameters such as:  
     - Boolean flag `notionEnabled` (true/false)  
     - Partner enabled flag and partner email address  
     - Telegram enabled flag and chat ID  
     - Email recipient addresses and subjects for notifications  
   - Connect outputs of `Weekly Cron` and `Manual Trigger` to `Set: User Config`.

3. **Configure Notion Task Retrieval:**  
   - Add an **If** node named `If: Notion Enabled?` that checks the `notionEnabled` flag from `Set: User Config`.  
   - On `true`, connect to a **Notion** node named `Notion: Get Tasks`.  
   - Configure the Notion node with credentials and specify the database ID or filters to retrieve tasks.  

4. **Extract Notion Tasks:**  
   - Add a **Function** node named `Function: Extract Notion Tasks`.  
   - Write JavaScript code to parse the Notion response data into an array of task objects with relevant fields (title, due date, etc.).  
   - Connect output from `Notion: Get Tasks` to this node.

5. **Sample Tasks Function:**  
   - Add a **Function** node named `Function: Sample Tasks`.  
   - Insert a predefined static array of sample task objects for use when Notion is disabled or manual testing is performed.  
   - Connect output of `Manual Trigger` to this node.

6. **Build Weekly Plan:**  
   - Add a **Function** node named `Function: Build Weekly Plan`.  
   - Write code to combine tasks from either `Function: Extract Notion Tasks` or `Function: Sample Tasks`.  
   - Format the output as an email-friendly string or HTML.  
   - Connect both `Function: Extract Notion Tasks` and `Function: Sample Tasks` outputs to this node.

7. **Send Email to User:**  
   - Add an **Email Send** node named `Email: Send to You`.  
   - Configure SMTP or email credentials.  
   - Set the recipient to the user's email address.  
   - Use expressions to set email subject and body from `Function: Build Weekly Plan`.  
   - Connect output of `Function: Build Weekly Plan` here.

8. **Partner Email Conditional:**  
   - Add an **If** node named `If: Partner Enabled?` checking the partner enabled flag.  
   - On `true`, connect to an **Email Send** node named `Email: Send to Partner`.  
   - Configure partner email address and SMTP credentials similarly.  
   - Connect output of `Function: Build Weekly Plan` to `If: Partner Enabled?`.

9. **Telegram Notification Conditional:**  
   - Add an **If** node named `If: Telegram Enabled?` checking the Telegram enabled flag.  
   - On `true`, connect to a **Telegram** node named `Telegram: Notify`.  
   - Configure Telegram Bot credentials and target chat ID.  
   - Connect output of `Function: Build Weekly Plan` to `If: Telegram Enabled?`.

10. **Error Handling Setup:**  
    - Add an **Error Trigger** node named `On Error`.  
    - Add an **Email Send** node named `On Error → Email Owner`.  
    - Configure to send error details to workflow owner’s email.  
    - Connect `On Error` node output to the email send node.

11. **Connect All Nodes Appropriately:**  
    - `Weekly Cron` → `Set: User Config`  
    - `Manual Trigger` → `Function: Sample Tasks` AND `Set: User Config`  
    - `Set: User Config` → `If: Notion Enabled?`  
    - `If: Notion Enabled?` (true) → `Notion: Get Tasks` → `Function: Extract Notion Tasks` → `Function: Build Weekly Plan`  
    - `Function: Sample Tasks` → `Function: Build Weekly Plan`  
    - `Function: Build Weekly Plan` → `Email: Send to You`, `If: Partner Enabled?`, and `If: Telegram Enabled?`  
    - `If: Partner Enabled?` (true) → `Email: Send to Partner`  
    - `If: Telegram Enabled?` (true) → `Telegram: Notify`  
    - `On Error` → `On Error → Email Owner`  

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                            |
|----------------------------------------------------------------------------------------------|------------------------------------------------------------|
| After importing, attach Email credentials and Notion/Telegram credentials if those are enabled. | Meta information in workflow JSON.                          |
| No secrets included in the exported JSON for security compliance.                            | Meta information in workflow JSON.                          |
| This workflow is designed for both household and business weekly task planning.              | Workflow description.                                       |
| Telegram notifications require a Bot Token and Chat ID set up in credentials prior to use.   | Telegram node configuration best practices.                |
| Notion integration requires an API token and database permissions configured in credentials. | Notion node documentation: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.notion/ |
| For troubleshooting, enable the error trigger node to receive email alerts on failures.      | Error Handling block description.                           |

---

**Disclaimer:** The provided text is exclusively derived from an automated n8n workflow. It strictly respects current content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.