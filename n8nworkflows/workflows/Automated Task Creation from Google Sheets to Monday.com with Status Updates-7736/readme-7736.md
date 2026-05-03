Automated Task Creation from Google Sheets to Monday.com with Status Updates

https://n8nworkflows.xyz/workflows/automated-task-creation-from-google-sheets-to-monday-com-with-status-updates-7736


# Automated Task Creation from Google Sheets to Monday.com with Status Updates

### 1. Workflow Overview

This workflow automates the creation of tasks in Monday.com based on entries in a Google Sheet. It targets project managers and teams who track tasks in Google Sheets but want to centralize task management in Monday.com without manual duplication. The workflow consists of three main logical blocks:

- **1.1 Input Reception and Filtering:** Checking the Google Sheet for new tasks marked as not yet added (`Added = No`).
- **1.2 Monday.com Task Creation:** Creating corresponding items on a specified Monday.com board and group.
- **1.3 Status Update Back to Google Sheets:** Marking processed tasks in the Google Sheet as added (`Added = Yes`) to avoid duplicates.

This design ensures seamless syncing from Google Sheets to Monday.com and prevents re-processing of tasks already transferred.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

- **Overview:**  
  This block reads the Google Sheet data, filtering rows where the `Added` column is set to `No`, indicating new tasks to be processed.

- **Nodes Involved:**  
  - `Get new Monday Tasks`

- **Node Details:**

  - **Get new Monday Tasks**  
    - **Type:** Google Sheets (Read operation)  
    - **Technical Role:** Reads rows from a Google Sheet filtered to only those tasks not yet added to Monday.com (`Added = No`).  
    - **Configuration:**  
      - Spreadsheet ID: `1KRiAUbZP77dC_9x5pqrvcQvaAkUsoPXkZOZvfU69ILM` (can be changed to user’s own sheet)  
      - Sheet Name: `Sheet1` (gid=0)  
      - Filtering: Only rows where `Added` equals `No` are fetched.  
    - **Input/Output:**  
      - Input: None (triggered externally or scheduled).  
      - Output: Array of task rows matching the filter.  
    - **Credentials:** Google Sheets OAuth2 account connected with appropriate permissions.  
    - **Edge Cases / Failure Types:**  
      - Authentication failure if OAuth token expired.  
      - Empty result if no new tasks found.  
      - Sheet or document ID invalid or inaccessible.  
      - Expression failure if the `Added` column is missing or renamed.

#### 2.2 Monday.com Task Creation

- **Overview:**  
  For each new task row obtained, a new task item is created on a specified Monday.com board and group.

- **Nodes Involved:**  
  - `Create Monday Task`

- **Node Details:**

  - **Create Monday Task**  
    - **Type:** Monday.com node (Create board item)  
    - **Technical Role:** Creates a new task/item on Monday.com with the task name taken from the Google Sheet’s `Task` column.  
    - **Configuration:**  
      - Board ID: `9865886546` (to be set by user)  
      - Group ID: `new_group29179` (to be set by user)  
      - Item Name: Expression `={{ $json.Task }}` (dynamic task name from Google Sheets)  
      - Additional fields: None configured by default but can be extended.  
    - **Input/Output:**  
      - Input: Individual task data from Google Sheets.  
      - Output: Monday.com API response with created item details.  
    - **Credentials:** Monday.com API Token connected under `Monday.com account`.  
    - **Edge Cases / Failure Types:**  
      - API authentication failure due to invalid or expired token.  
      - Invalid board or group IDs causing API errors.  
      - Rate limiting or API timeouts.  
      - Missing `Task` field in input data causing empty item names.  
    - **Sub-workflow:** None.

#### 2.3 Status Update Back to Google Sheets

- **Overview:**  
  After a Monday.com task is created, this block updates the original Google Sheet row, marking the task as added (`Added = Yes`) to prevent re-processing.

- **Nodes Involved:**  
  - `Mark row as Completed`

- **Node Details:**

  - **Mark row as Completed**  
    - **Type:** Google Sheets (Append or Update operation)  
    - **Technical Role:** Updates the row in the Google Sheet matching the task name, setting the `Added` column to `Yes`.  
    - **Configuration:**  
      - Spreadsheet ID and Sheet name same as in input block.  
      - Matching column: `Task` is used to find the exact row to update.  
      - Columns updated: `Task` (retained), `Added` set to `Yes`.  
      - Mapping mode: Defined below for precise control.  
    - **Input/Output:**  
      - Input: Task data from Monday.com creation node.  
      - Output: Confirmation of updated row.  
    - **Credentials:** Same Google Sheets OAuth2 credential as input.  
    - **Edge Cases / Failure Types:**  
      - Row not found if task name changed or duplicates exist.  
      - Authentication or permission errors.  
      - Concurrent updates causing conflicts.  
      - Data type mismatches if column schemas change.

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                    | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                          |
|----------------------|---------------------|----------------------------------|------------------------|------------------------|----------------------------------------------------------------------------------------------------|
| Sticky Note63        | Sticky Note         | Workflow purpose and summary     | -                      | -                      | # ✅ Google Sheets to Monday.com Task Creator. This workflow checks a **Google Sheet** for new tasks (marked `Added = No`) and automatically creates them in a **Monday.com board**. Once added, the workflow updates the sheet to mark the task as `Added = Yes`. |
| Sticky Note21        | Sticky Note         | Setup instructions for users     | -                      | -                      | ## ⚙️ Setup Instructions. 1️⃣ Prepare Your Google Sheet (with link). 2️⃣ Connect Monday.com Node with API token and credential setup. Contact info included. |
| Sticky Note24        | Sticky Note         | Google Sheets preparation details| -                      | -                      | ### 1️⃣ Prepare Your Google Sheet. Copy template, add data, start `Added = No`. Connect Google Sheets OAuth2 credential in n8n. |
| Sticky Note67        | Sticky Note         | Monday.com API connection details| -                      | -                      | ### 2️⃣ Connect Monday.com Node. Obtain API token, create credential, configure node with Board and Group IDs. Link to docs provided. |
| Get new Monday Tasks | Google Sheets       | Reads new tasks from Google Sheet| -                      | Create Monday Task      |                                                                                                    |
| Create Monday Task   | Monday.com          | Creates tasks on Monday.com board| Get new Monday Tasks    | Mark row as Completed   |                                                                                                    |
| Mark row as Completed| Google Sheets       | Updates Google Sheet task status | Create Monday Task      | -                      |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets OAuth2 Credential:**  
   - In n8n, navigate to **Credentials → New → Google Sheets (OAuth2)**.  
   - Authenticate with your Google account and grant access to your sheets.

2. **Create Monday.com API Credential:**  
   - Go to Monday.com Admin → API, generate a Personal API token.  
   - In n8n, create a new **Monday.com API** credential, paste the token, and save.

3. **Add Node: Get new Monday Tasks (Google Sheets node)**  
   - Set **Operation** to "Read".  
   - Specify your Google Sheet by Spreadsheet ID (e.g., `1KRiAUbZP77dC_9x5pqrvcQvaAkUsoPXkZOZvfU69ILM`).  
   - Set Sheet Name to your target sheet (e.g., `Sheet1` or gid=0).  
   - Add a filter to only fetch rows where `Added` column equals `No`.  
   - Use the Google Sheets OAuth2 credential created earlier.

4. **Add Node: Create Monday Task (Monday.com node)**  
   - Set **Resource** to `boardItem`.  
   - Set **Operation** to `create`.  
   - Use your Monday.com API credential.  
   - Set Board ID (e.g., `9865886546`) and Group ID (e.g., `new_group29179`).  
   - For Item Name, set expression to `={{ $json.Task }}` to use the task name from the previous node.  
   - Connect output of “Get new Monday Tasks” node to this node.

5. **Add Node: Mark row as Completed (Google Sheets node)**  
   - Set **Operation** to `appendOrUpdate`.  
   - Use the same Spreadsheet ID and Sheet Name as the first node.  
   - Set **Matching Columns** to `Task` to identify the row to update.  
   - Define columns to update: set `Added` to `Yes` and pass through the `Task` name unchanged.  
   - Use the same Google Sheets OAuth2 credential.  
   - Connect output of “Create Monday Task” node to this node.

6. **Connect Nodes Sequentially:**  
   - `Get new Monday Tasks` → `Create Monday Task` → `Mark row as Completed`.

7. **Optional: Add Sticky Notes** (for documentation inside n8n):  
   - Add notes describing the workflow purpose and setup instructions for Google Sheets and Monday.com credentials.  
   - Include links to Google Sheet templates and Monday.com API docs.

8. **Test the Workflow:**  
   - Ensure you have tasks with `Added = No` in your Google Sheet.  
   - Run the workflow manually or set up a trigger.  
   - Verify tasks are created in Monday.com.  
   - Confirm the Google Sheet rows update to `Added = Yes`.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                  | Context or Link                                                                                                    |
|-----------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------|
| Google Sheet template for task input with predefined columns and example data.                                                                | [Google Sheet Template](https://docs.google.com/spreadsheets/d/1KRiAUbZP77dC_9x5pqrvcQvaAkUsoPXkZOZvfU69ILM/edit?gid=876214427#gid=876214427) |
| Monday.com API token generation guide and authentication documentation.                                                                       | [Monday.com API Token Generation](https://developer.monday.com/api-reference/docs/authentication)                 |
| Contact for customization support, including mapping additional fields or syncing statuses.                                                  | Email: robert@ynteractive.com; LinkedIn: [Robert Breen](https://www.linkedin.com/in/robert-breen-29429625/); Website: [ynteractive.com](https://ynteractive.com) |

---

**Disclaimer:**  
The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with in-force content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.