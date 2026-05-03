Onboard employees from Google Forms with Slack, Jira, and GitHub access

https://n8nworkflows.xyz/workflows/onboard-employees-from-google-forms-with-slack--jira--and-github-access-14221


# Onboard employees from Google Forms with Slack, Jira, and GitHub access

# 1. Workflow Overview

This workflow automates employee onboarding from a Google Form response stored in Google Sheets. When a new form response arrives, it enriches the submitted data with department-specific routing rules, checks whether onboarding was already completed, and if not, provisions collaboration access across Slack, GitHub, and Jira before marking the record as completed in the source sheet.

Typical use cases:
- HR-led onboarding triggered from a Google Form
- IT access provisioning for new hires
- Department-based routing for collaboration tools
- Tracking onboarding completion directly in the form response sheet

## 1.1 Input Reception and Configuration Mapping
The workflow starts when a new row is added to a Google Sheet receiving Google Form responses. It then maps department/team values into operational targets such as Slack channel IDs, Jira project keys, Jira component IDs, GitHub repositories, and a software-department flag.

## 1.2 Duplicate/Completed Check
The workflow checks whether the submitted employee record already has `Status = Completed`. If yes, it does not re-run provisioning and instead notifies an admin in Slack.

## 1.3 Core Provisioning
If onboarding has not been completed, the workflow joins the automation’s Slack identity to the target department channel, then branches based on whether the employee belongs to the Software department.

## 1.4 Software-Specific GitHub Access
If the employee is in Software, the workflow grants GitHub repository access based on the submitted team.

## 1.5 Jira Invitation and Onboarding Tracking
The workflow invites the user to Jira, creates a Jira onboarding task in the mapped project with an optional team component, and finally updates the Google Sheet row to `Completed`.

---

# 2. Block-by-Block Analysis

## Block 1 — Input Reception and Configuration Mapping

### Overview
This block captures new form submissions from Google Sheets and converts human-readable form values such as department and team into system-specific routing values. It is the foundation for all downstream branching and provisioning logic.

### Nodes Involved
- New Google Form Response
- Map department and team configuration

### Node Details

#### 1. New Google Form Response
- **Type and technical role:** `Google Sheets Trigger` node. It polls a spreadsheet for newly added rows.
- **Configuration choices:**
  - Event: `rowAdded`
  - Polling frequency: every minute
  - Sheet name: `Form Responses 1`
  - Document ID: placeholder `YOUR_GOOGLE_SHEET_ID_HERE`
- **Key expressions or variables used:** None in node configuration, but downstream nodes assume columns such as:
  - `Full Name`
  - `Email`
  - `Department`
  - `Team`
  - `GitHub Username`
  - `Status`
- **Input and output connections:**
  - Input: none, this is an entry point
  - Output: `Map department and team configuration`
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires Google Sheets Trigger OAuth2 credentials
- **Edge cases or potential failure types:**
  - Wrong spreadsheet ID or sheet name
  - OAuth token expiration
  - Polling delays
  - Missing expected columns in incoming rows
  - Duplicate rows if the source sheet is manually edited in ways that resemble new entries

#### 2. Map department and team configuration
- **Type and technical role:** `Code` node. It transforms incoming form data into operational routing metadata.
- **Configuration choices:**
  - Uses a JavaScript object named `config` with:
    - `departments`: maps department names to Slack channel IDs and Jira project keys
    - `componentMap`: maps team names to Jira component IDs
    - `teams`: maps team names to GitHub repository names
  - Builds normalized output fields:
    - `targetSlack`
    - `targetJira`
    - `targetComponentIds`
    - `targetRepo`
    - `targetGithubUser`
    - `isSoftware`
- **Key expressions or variables used:**
  - `const item = $input.item.json;`
  - `item.Department`
  - `item.Team`
  - `item["GitHub Username"]`
  - `item.Email`
  - Fallback defaults:
    - general Slack channel
    - default Jira project
    - general GitHub repo
- **Input and output connections:**
  - Input: `New Google Form Response`
  - Output: `Check if onboarding already completed`
- **Version-specific requirements:**
  - `typeVersion: 2`
  - JavaScript syntax must be compatible with the n8n Code node runtime
- **Edge cases or potential failure types:**
  - Department or team not found in config maps
  - Missing `GitHub Username`
  - Inconsistent case or spelling in form values, e.g. `frontend` vs `FrontEnd`
  - Invalid Slack/Jira/GitHub placeholder values left unchanged
  - Team with no Jira component mapping results in an empty component array
- **Sub-workflow reference:** None

---

## Block 2 — Duplicate/Completed Check

### Overview
This block prevents duplicate provisioning by checking whether the Google Sheet row is already marked completed. If it is, a Slack notification is sent to an admin instead of re-running the rest of onboarding.

### Nodes Involved
- Check if onboarding already completed
- Notify admin if already processed

### Node Details

#### 3. Check if onboarding already completed
- **Type and technical role:** `IF` node. It performs a strict equality test on the `Status` field.
- **Configuration choices:**
  - Condition: `{{$json.Status}} equals Completed`
  - Strict validation enabled
  - Case-sensitive comparison
- **Key expressions or variables used:**
  - `={{ $json.Status }}`
- **Input and output connections:**
  - Input: `Map department and team configuration`
  - True output: `Notify admin if already processed`
  - False output: `Add user to department Slack channel`
- **Version-specific requirements:**
  - `typeVersion: 2.2`
- **Edge cases or potential failure types:**
  - Status values like `completed`, `COMPLETE`, or trailing spaces will not match
  - Missing `Status` field routes the item to the “not completed” branch
  - If historical rows use different wording, duplicate provisioning may occur
- **Sub-workflow reference:** None

#### 4. Notify admin if already processed
- **Type and technical role:** `Slack` node. Sends a direct message or user-targeted notification to an admin account.
- **Configuration choices:**
  - Target selection mode: `user`
  - User ID placeholder: `YOUR_SLACK_ADMIN_USER_ID_HERE`
  - Message text includes employee full name and email
- **Key expressions or variables used:**
  - `{{ $('Map department and team configuration').item.json['Full Name'] }}`
  - `{{ $('Map department and team configuration').item.json.Email }}`
- **Input and output connections:**
  - Input: true branch from `Check if onboarding already completed`
  - Output: none
- **Version-specific requirements:**
  - `typeVersion: 2.3`
  - Requires Slack API credentials with permission to message the selected user
- **Edge cases or potential failure types:**
  - Invalid admin user ID
  - Slack app missing DM-related scope or messaging scope
  - Message formatting issues if name/email fields are absent
- **Sub-workflow reference:** None

---

## Block 3 — Core Provisioning

### Overview
This block begins actual onboarding for records not already completed. It first joins the automation to the appropriate department Slack channel, then decides whether GitHub provisioning is required.

### Nodes Involved
- Add user to department Slack channel
- Check if department is software

### Node Details

#### 5. Add user to department Slack channel
- **Type and technical role:** `Slack` node using the channel resource. It performs a `join` operation for the configured Slack channel.
- **Configuration choices:**
  - Resource: `channel`
  - Operation: `join`
  - Channel ID: `={{ $json.targetSlack }}`
  - Error handling: `continueRegularOutput`
- **Key expressions or variables used:**
  - `={{ $json.targetSlack }}`
- **Input and output connections:**
  - Input: false branch from `Check if onboarding already completed`
  - Output: `Check if department is software`
- **Version-specific requirements:**
  - `typeVersion: 2.3`
- **Important behavioral note:**
  - As configured, this joins the **Slack app/bot/authenticated identity** to a channel. It does **not** invite the employee to Slack, nor does it add a specific user to a channel.
- **Edge cases or potential failure types:**
  - Invalid channel ID
  - Bot not permitted to join private channels
  - Slack app missing channel scopes
  - Because `continueRegularOutput` is enabled, downstream processing continues even if Slack join fails
- **Sub-workflow reference:** None

#### 6. Check if department is software
- **Type and technical role:** `IF` node. Branches based on a boolean flag computed in the Code node.
- **Configuration choices:**
  - Condition: boolean is true for `isSoftware`
  - Uses value from the Code node, not directly from current JSON
- **Key expressions or variables used:**
  - `={{ $('Map department and team configuration').item.json.isSoftware }}`
- **Input and output connections:**
  - Input: `Add user to department Slack channel`
  - True output: `Add user to GitHub repository`
  - False output: `Invite user to Jira`
- **Version-specific requirements:**
  - `typeVersion: 2.2`
- **Edge cases or potential failure types:**
  - Department spelling mismatch in the Code node makes `isSoftware` false
  - If the Code node logic is modified, this branch may behave unexpectedly
- **Sub-workflow reference:** None

---

## Block 4 — Software-Specific GitHub Access

### Overview
This block is only executed for employees in the Software department. It grants repository collaboration access to the submitted GitHub username using the configured repository mapping.

### Nodes Involved
- Add user to GitHub repository

### Node Details

#### 7. Add user to GitHub repository
- **Type and technical role:** `HTTP Request` node authenticated with GitHub credentials. It calls the GitHub REST API to add a collaborator to a repository.
- **Configuration choices:**
  - Method: `PUT`
  - URL pattern:
    - `https://api.github.com/repos/YOUR_ORG_NAME/{{targetRepo}}/collaborators/{{GitHub Username}}`
  - Header:
    - `Accept: application/vnd.github.v3+json`
  - Body:
    - `permission: push`
  - Authentication:
    - predefined credential type `githubApi`
- **Key expressions or variables used:**
  - `{{ $('Map department and team configuration').item.json.targetRepo }}`
  - `{{ $('Map department and team configuration').item.json['GitHub Username'] }}`
- **Input and output connections:**
  - Input: true branch from `Check if department is software`
  - Output: `Invite user to Jira`
- **Version-specific requirements:**
  - `typeVersion: 4.3`
  - Requires GitHub credentials with sufficient org/repository admin permissions
- **Edge cases or potential failure types:**
  - Invalid GitHub organization name
  - Invalid repository name
  - Missing or incorrect GitHub username
  - Repository invitation restrictions in the org
  - Credential lacks permission to add collaborators
  - User may receive an invitation instead of immediate access depending on org settings
- **Sub-workflow reference:** None

---

## Block 5 — Jira Invitation, Task Creation, and Source Update

### Overview
This block invites the user to Jira, creates a Jira onboarding task in the mapped project, and updates the original Google Sheet record to show completion. It finalizes the onboarding lifecycle tracked by the workflow.

### Nodes Involved
- Invite user to Jira
- Create onboarding task in Jira
- Update Google Sheet status to completed

### Node Details

#### 8. Invite user to Jira
- **Type and technical role:** `HTTP Request` node authenticated with Jira credentials. It sends a user invitation request to an Atlassian site.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://YOUR_SITE.atlassian.net`
  - JSON body:
    - `emailAddress`
    - `products: ["jira-software"]`
  - Authentication:
    - predefined credential type `jiraSoftwareCloudApi`
- **Key expressions or variables used:**
  - `{{ $('Map department and team configuration').item.json.Email }}`
- **Input and output connections:**
  - Input:
    - false branch from `Check if department is software`
    - output from `Add user to GitHub repository`
  - Output: `Create onboarding task in Jira`
- **Version-specific requirements:**
  - `typeVersion: 4.3`
- **Important behavioral note:**
  - The URL shown is only the base site URL. In practice, Jira/Atlassian user invitation usually requires a specific admin API endpoint, not just the root site URL. This node likely requires adjustment before production use.
- **Edge cases or potential failure types:**
  - Wrong endpoint
  - Atlassian admin permissions missing
  - Email already invited or already exists
  - Site-level user provisioning restrictions
  - Rate limits or validation errors on request body
- **Sub-workflow reference:** None

#### 9. Create onboarding task in Jira
- **Type and technical role:** `Jira` node. Creates an issue representing the onboarding task.
- **Configuration choices:**
  - Project is dynamically selected from `targetJira`
  - Summary: `Onboarding: <employee email>`
  - Issue type: `Task` with ID `10014`
  - Additional fields:
    - Description includes department
    - Component IDs array populated from first entry in `targetComponentIds`
- **Key expressions or variables used:**
  - `={{ $('Map department and team configuration').item.json.targetJira }}`
  - `{{ $('Map department and team configuration').item.json.Email }}`
  - `{{ $('Map department and team configuration').item.json.Department }}`
  - `={{ [$('Map department and team configuration').item.json.targetComponentIds[0]] }}`
- **Input and output connections:**
  - Input: `Invite user to Jira`
  - Output: `Update Google Sheet status to completed`
- **Version-specific requirements:**
  - `typeVersion: 1`
  - Requires Jira Software Cloud credentials
  - Project field is configured in “id” mode but receives a mapped project key-like value; this should be validated in the target n8n/Jira setup
- **Edge cases or potential failure types:**
  - Invalid project value
  - Issue type ID `10014` may differ between Jira instances
  - Component ID may be invalid or empty
  - Permissions may allow issue creation in some projects but not others
- **Sub-workflow reference:** None

#### 10. Update Google Sheet status to completed
- **Type and technical role:** `Google Sheets` node. Updates the matching row in the responses sheet to set onboarding status.
- **Configuration choices:**
  - Operation: `update`
  - Sheet: `Form Responses 1`
  - Document ID: placeholder `YOUR_GOOGLE_SHEET_ID_HERE`
  - Matching column: `Email`
  - Updated values:
    - `Email`
    - `Status = Completed`
- **Key expressions or variables used:**
  - `{{ $('Map department and team configuration').item.json.Email }}`
- **Input and output connections:**
  - Input: `Create onboarding task in Jira`
  - Output: none
- **Version-specific requirements:**
  - `typeVersion: 4.7`
  - Requires Google Sheets OAuth2 credentials with edit access
- **Edge cases or potential failure types:**
  - No matching row found by email
  - Duplicate email rows may update the wrong row or multiple rows depending on backend behavior
  - Spreadsheet ID or sheet mismatch
  - If prior provisioning partially failed but Jira task succeeded, the row may still be marked completed
- **Sub-workflow reference:** None

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Workflow Explanation | Sticky Note | Documents overall purpose, setup, requirements, and customization |  |  | ## Overview<br>This workflow automates employee onboarding using data submitted through a Google Form. Based on the employee's department and role, it provisions access to Slack, Jira, and GitHub, and tracks onboarding status.<br><br>## Who's it for<br>- HR teams managing employee onboarding<br>- IT administrators handling access provisioning<br>- Companies using Google Forms for new hire intake<br>- Teams using Slack, Jira, and GitHub for collaboration<br><br>## How it works<br>1. A new response is submitted via Google Forms<br>2. The workflow checks if the employee has already been processed<br>3. If already completed, the admin is notified<br>4. If not:<br>   - User is added to the appropriate Slack channel<br>   - If in the Software department, GitHub access is granted<br>   - User is invited to Jira<br>   - A Jira onboarding task is created<br>5. The Google Sheet is updated to mark onboarding as completed<br><br>## Setup Steps<br>1. **Google Sheets**: Connect your account and update `YOUR_GOOGLE_SHEET_ID_HERE` in both the trigger and update nodes<br>2. **Slack**: Connect your account and update:<br>   - Department channel IDs in the Code node config<br>   - Admin user ID in "Notify admin" node<br>3. **Jira**: Connect your account and update:<br>   - Project keys in the Code node config<br>   - Component IDs in the Code node config<br>   - Your site URL in the "Invite user to Jira" node<br>4. **GitHub**: Connect your account and update:<br>   - Organization name in the HTTP node URL<br>   - Repository names in the Code node config<br><br>## Requirements<br>- Google Sheets with form responses<br>- Slack workspace with appropriate channels<br>- Jira Software Cloud instance<br>- GitHub organization with repositories<br><br>## Customization<br>- Modify department mappings in the Code node to match your org structure<br>- Update Slack channel IDs for each department<br>- Adjust GitHub repository names for each team<br>- Configure Jira component IDs for different roles |
| Group: Sorting Office | Sticky Note | Visual grouping for validation and preprocessing |  |  | ### Data processing and validation<br>Processes form input and checks if onboarding was already completed. |
| Group: Digital Onboarding | Sticky Note | Visual grouping for provisioning and onboarding actions |  |  | ### User provisioning and onboarding<br>Creates accounts, assigns access, and sets up onboarding tasks. |
| New Google Form Response | Google Sheets Trigger | Starts workflow when a new form response row is added |  | Map department and team configuration | ### Data processing and validation<br>Processes form input and checks if onboarding was already completed. |
| Map department and team configuration | Code | Maps department/team values to Slack, Jira, and GitHub targets | New Google Form Response | Check if onboarding already completed | ### Data processing and validation<br>Processes form input and checks if onboarding was already completed. |
| Check if onboarding already completed | IF | Prevents duplicate onboarding based on `Status` | Map department and team configuration | Notify admin if already processed; Add user to department Slack channel | ### Data processing and validation<br>Processes form input and checks if onboarding was already completed. |
| Notify admin if already processed | Slack | Alerts an admin that the employee was already processed | Check if onboarding already completed |  | ### Data processing and validation<br>Processes form input and checks if onboarding was already completed. |
| Add user to department Slack channel | Slack | Joins the configured Slack identity to a department channel | Check if onboarding already completed | Check if department is software | ### User provisioning and onboarding<br>Creates accounts, assigns access, and sets up onboarding tasks. |
| Check if department is software | IF | Routes software employees through GitHub provisioning | Add user to department Slack channel | Add user to GitHub repository; Invite user to Jira | ### User provisioning and onboarding<br>Creates accounts, assigns access, and sets up onboarding tasks. |
| Add user to GitHub repository | HTTP Request | Grants GitHub repo collaborator access for software employees | Check if department is software | Invite user to Jira | ### User provisioning and onboarding<br>Creates accounts, assigns access, and sets up onboarding tasks. |
| Invite user to Jira | HTTP Request | Sends Jira/Atlassian invitation request | Check if department is software; Add user to GitHub repository | Create onboarding task in Jira | ### User provisioning and onboarding<br>Creates accounts, assigns access, and sets up onboarding tasks. |
| Create onboarding task in Jira | Jira | Creates a Jira onboarding task in the mapped project | Invite user to Jira | Update Google Sheet status to completed | ### User provisioning and onboarding<br>Creates accounts, assigns access, and sets up onboarding tasks. |
| Update Google Sheet status to completed | Google Sheets | Marks onboarding as completed in the source sheet | Create onboarding task in Jira |  | ### User provisioning and onboarding<br>Creates accounts, assigns access, and sets up onboarding tasks. |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - Name it: **Onboard employees from Google Forms with Slack, Jira, and GitHub access**.

2. **Add a Google Sheets Trigger node**
   - Node type: **Google Sheets Trigger**
   - Name: **New Google Form Response**
   - Set:
     - Event: **Row Added**
     - Polling: **Every Minute**
     - Document ID: your Google Sheet ID
     - Sheet Name: `Form Responses 1`
   - Connect Google credentials using a **Google Sheets Trigger OAuth2** credential.
   - Ensure your sheet contains columns at least equivalent to:
     - `Full Name`
     - `Email`
     - `Department`
     - `Team`
     - `GitHub Username`
     - `Status`

3. **Add a Code node**
   - Node type: **Code**
   - Name: **Map department and team configuration**
   - Connect it after the trigger.
   - Configure JavaScript logic that:
     - Reads the incoming row
     - Maps `Department` to:
       - a Slack channel ID
       - a Jira project key or project identifier
     - Maps `Team` to:
       - a Jira component ID
       - a GitHub repository name
     - Produces:
       - `targetSlack`
       - `targetJira`
       - `targetComponentIds`
       - `targetRepo`
       - `targetGithubUser`
       - `isSoftware`
   - Use fallback defaults for unknown departments/teams.
   - Example logic to implement:
     - Departments:
       - `Software` → Slack channel + Jira project
       - `Marketing` → Slack channel + Jira project
       - `Finance` → Slack channel + Jira project
       - `Project` → Slack channel + Jira project
     - Team-to-component map:
       - `FrontEnd`, `BackEnd`, `Mobile`, `UI/UX`, `Tester`, `Project Manager`, `Business Analyst`, `Scrum Master`
     - Team-to-repo map:
       - e.g. frontend repo, backend repo, mobile repo, design repo, QA repo

4. **Add an IF node to prevent duplicate processing**
   - Node type: **IF**
   - Name: **Check if onboarding already completed**
   - Connect it after the Code node.
   - Condition:
     - Left value: `{{$json.Status}}`
     - Operator: **equals**
     - Right value: `Completed`
   - Keep in mind this is case-sensitive.

5. **Add a Slack node for duplicate alerts**
   - Node type: **Slack**
   - Name: **Notify admin if already processed**
   - Connect it to the **true** output of the IF node.
   - Configure:
     - Operation to send a message to a **user**
     - Target user: your Slack admin user ID
     - Message text including:
       - employee full name
       - employee email
       - note that status is already completed
   - Add Slack credentials with the appropriate scopes.

6. **Add a Slack node for department channel join**
   - Node type: **Slack**
   - Name: **Add user to department Slack channel**
   - Connect it to the **false** output of the duplicate-check IF node.
   - Configure:
     - Resource: **Channel**
     - Operation: **Join**
     - Channel ID: `{{$json.targetSlack}}`
   - In node settings, set **On Error** to **Continue Regular Output**.
   - Note: this joins the authenticated Slack app/user to the channel. If your real requirement is to invite the employee, you will need a different Slack operation or API call.

7. **Add a second IF node for software branching**
   - Node type: **IF**
   - Name: **Check if department is software**
   - Connect it after the Slack join node.
   - Configure a boolean condition:
     - Left value: `{{ $('Map department and team configuration').item.json.isSoftware }}`
     - Operator: **is true**

8. **Add a GitHub HTTP Request node**
   - Node type: **HTTP Request**
   - Name: **Add user to GitHub repository**
   - Connect it to the **true** output of the software-check IF node.
   - Configure:
     - Method: **PUT**
     - URL:
       - `https://api.github.com/repos/YOUR_ORG_NAME/{{ $('Map department and team configuration').item.json.targetRepo }}/collaborators/{{ $('Map department and team configuration').item.json['GitHub Username'] }}`
     - Authentication: **Predefined Credential Type**
     - Credential type: **GitHub API**
     - Headers:
       - `Accept: application/vnd.github.v3+json`
     - Body:
       - `permission: push`
   - Add GitHub credentials with permission to manage repository collaborators.

9. **Add a Jira invitation HTTP Request node**
   - Node type: **HTTP Request**
   - Name: **Invite user to Jira**
   - Connect it to:
     - the **false** output of **Check if department is software**
     - the output of **Add user to GitHub repository**
   - Configure:
     - Method: **POST**
     - Authentication: **Predefined Credential Type**
     - Credential type: **Jira Software Cloud API**
     - URL: your Atlassian admin invitation endpoint or site endpoint
     - Body format: **JSON**
     - JSON body should include:
       - `emailAddress`: employee email
       - `products`: `["jira-software"]`
   - Important: the template uses the site root URL, but in a working implementation you should replace it with the actual Atlassian invitation endpoint supported by your environment.

10. **Add a Jira node to create the onboarding task**
    - Node type: **Jira**
    - Name: **Create onboarding task in Jira**
    - Connect it after **Invite user to Jira**.
    - Configure:
      - Operation: **Create Issue**
      - Project: dynamic expression from `targetJira`
      - Summary: `Onboarding: <employee email>`
      - Issue Type: **Task**
      - Additional fields:
        - Description mentioning the employee department
        - Component IDs from the mapped team component
    - Validate:
      - the issue type ID exists in your Jira instance
      - the project value format matches what the node expects
      - the component IDs are valid for the selected project

11. **Add a Google Sheets node to mark completion**
    - Node type: **Google Sheets**
    - Name: **Update Google Sheet status to completed**
    - Connect it after the Jira issue creation node.
    - Configure:
      - Operation: **Update**
      - Document ID: same Google Sheet ID as the trigger
      - Sheet Name: `Form Responses 1`
      - Matching column: `Email`
      - Fields to update:
        - `Email` = employee email
        - `Status` = `Completed`
   - Use Google Sheets OAuth2 credentials with write access.

12. **Add documentation sticky notes**
    - Add one large sticky note for overall workflow explanation, including:
      - purpose
      - setup steps
      - system requirements
      - customization notes
    - Add one sticky note around the intake/validation nodes:
      - “Data processing and validation”
    - Add one sticky note around the provisioning nodes:
      - “User provisioning and onboarding”

13. **Connect the nodes in this exact order**
   - `New Google Form Response` → `Map department and team configuration`
   - `Map department and team configuration` → `Check if onboarding already completed`
   - True branch → `Notify admin if already processed`
   - False branch → `Add user to department Slack channel`
   - `Add user to department Slack channel` → `Check if department is software`
   - True branch → `Add user to GitHub repository` → `Invite user to Jira`
   - False branch → `Invite user to Jira`
   - `Invite user to Jira` → `Create onboarding task in Jira`
   - `Create onboarding task in Jira` → `Update Google Sheet status to completed`

14. **Configure credentials**
   - **Google Sheets Trigger OAuth2** for the trigger
   - **Google Sheets OAuth2** for the update node
   - **Slack API** for both Slack nodes
   - **Jira Software Cloud API** for the Jira HTTP Request and Jira issue node
   - **GitHub API** for the GitHub HTTP Request node

15. **Replace all placeholders**
   - `YOUR_GOOGLE_SHEET_ID_HERE`
   - `YOUR_SOFTWARE_SLACK_CHANNEL_ID`
   - `YOUR_MARKETING_SLACK_CHANNEL_ID`
   - `YOUR_FINANCE_SLACK_CHANNEL_ID`
   - `YOUR_PROJECT_SLACK_CHANNEL_ID`
   - `YOUR_GENERAL_SLACK_CHANNEL_ID`
   - `YOUR_SOFTWARE_JIRA_PROJECT_KEY`
   - `YOUR_MKT_JIRA_PROJECT_KEY`
   - `YOUR_FIN_JIRA_PROJECT_KEY`
   - `YOUR_PROJ_JIRA_PROJECT_KEY`
   - `YOUR_DEFAULT_JIRA_PROJECT_KEY`
   - `YOUR_FRONTEND_COMPONENT_ID`
   - `YOUR_BACKEND_COMPONENT_ID`
   - `YOUR_MOBILE_COMPONENT_ID`
   - `YOUR_UIUX_COMPONENT_ID`
   - `YOUR_TESTER_COMPONENT_ID`
   - `YOUR_PM_COMPONENT_ID`
   - `YOUR_BA_COMPONENT_ID`
   - `YOUR_SM_COMPONENT_ID`
   - `YOUR_ORG_NAME`
   - `YOUR_SITE`
   - `YOUR_SLACK_ADMIN_USER_ID_HERE`
   - Repository names for each team

16. **Test with sample rows**
   - Test case 1: `Status = Completed`
     - Expected: only admin Slack notification
   - Test case 2: non-software employee
     - Expected: Slack join → Jira invite → Jira task → Google Sheet update
   - Test case 3: software employee with valid GitHub username
     - Expected: Slack join → GitHub collaborator request → Jira invite → Jira task → Google Sheet update
   - Test case 4: unknown department/team
     - Expected: fallback Slack/Jira/GitHub targets used

17. **Validate operational assumptions**
   - Confirm whether Slack should:
     - join the bot to a channel, or
     - actually invite the employee
   - Confirm the Atlassian invitation endpoint
   - Confirm project keys vs project IDs expected by the Jira node
   - Confirm issue type ID and component IDs for each Jira project

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| The workflow includes a built-in overview sticky note describing purpose, intended audience, setup steps, requirements, and customization points. | Internal workflow documentation |
| Google Sheets is used as both the trigger source and the completion tracker. The same spreadsheet ID must be configured in both Google Sheets nodes. | Google Sheets configuration |
| The Slack provisioning step, as configured, joins the authenticated Slack entity to a channel rather than inviting the employee. Review this carefully before production use. | Slack behavior note |
| The Jira invitation HTTP request appears incomplete as a production-ready Atlassian invitation call and should be replaced with the correct endpoint for your environment. | Jira/Atlassian API configuration |
| Department mappings, Jira component IDs, GitHub repositories, and Slack channel IDs are centralized in the Code node, making that node the main customization point. | Workflow customization |