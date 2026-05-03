Create Daily Task Status Snapshots from Monday.com to Google Sheets

https://n8nworkflows.xyz/workflows/create-daily-task-status-snapshots-from-monday-com-to-google-sheets-7739


# Create Daily Task Status Snapshots from Monday.com to Google Sheets

### 1. Workflow Overview

This workflow is designed to create daily snapshots of task statuses from a Monday.com board and log them into a Google Sheet. The primary use case is to maintain an automated daily record of project progress and task statuses for reporting or tracking purposes.

The workflow is logically divided into the following blocks:

- **1.1 Monday.com Data Retrieval:** Connects to Monday.com API and fetches all tasks from a specified board and group.
- **1.2 Data Mapping and Transformation:** Maps and formats the retrieved Monday.com data fields to a simplified structure suitable for Google Sheets.
- **1.3 Date Annotation:** Generates the current date to timestamp each snapshot entry.
- **1.4 Data Merging and Appending:** Merges task data with the date and appends the combined records into a Google Sheet.
- **1.5 Setup and Documentation:** Provides user instructions and references for configuration, including API credentials and Google Sheets setup.

---

### 2. Block-by-Block Analysis

#### 2.1 Monday.com Data Retrieval

- **Overview:**  
  This block connects to the Monday.com API using a stored credential and retrieves all items (tasks) from a specified board and group.

- **Nodes Involved:**  
  - Get many items2

- **Node Details:**

  - **Get many items2**  
    - Type: Monday.com node (API integration)  
    - Configuration:  
      - Board ID: `9865886546` (target Monday.com board)  
      - Group ID: `new_group29179` (specific group within the board)  
      - Resource: `boardItem` (fetching board items/tasks)  
      - Operation: `getAll` (retrieve all items)  
    - Input: None (start node)  
    - Output: List of tasks with detailed fields including column values  
    - Credentials: Uses configured Monday.com API credentials (Personal API Token)  
    - Edge Cases / Potential Failures:  
      - API authentication failures (invalid or expired token)  
      - Rate limiting or API downtime  
      - Empty board/group resulting in no data  
    - Version: n8n Monday.com node v1  

#### 2.2 Data Mapping and Transformation

- **Overview:**  
  This block processes the raw Monday.com data to extract relevant fields and restructure them for later insertion into Google Sheets.

- **Nodes Involved:**  
  - Map Fields

- **Node Details:**

  - **Map Fields**  
    - Type: Set node (data transformation)  
    - Configuration:  
      - Maps Monday.com item fields to simpler keys:  
        - `Task Name` set to the item’s `name` field  
        - `Status` extracted from the second column value’s `text` property (`column_values[1].text`)  
    - Key Expressions:  
      - `={{ $json.name }}` for task name  
      - `={{ $json.column_values[1].text }}` for status text  
    - Input: Output from "Get many items2" node  
    - Output: Objects with `Task Name` and `Status` fields only  
    - Edge Cases / Potential Failures:  
      - Assumes `column_values[1]` exists and contains `.text` (may fail if the board’s column structure changes)  
      - Missing or null fields in tasks  
    - Version: Set node v3.4  

#### 2.3 Date Annotation

- **Overview:**  
  Generates the current date in ISO format to timestamp the daily task snapshot.

- **Nodes Involved:**  
  - Today's Date2

- **Node Details:**

  - **Today's Date2**  
    - Type: Code node (JavaScript)  
    - Configuration:  
      - Returns a JSON object with a nested property `badges.today` containing the current date in `YYYY-MM-DD` format.  
      - Code snippet:
        ```js
        return [
          {
            json: {
              badges: {
                today: new Date().toISOString().split('T')[0]
              }
            }
          }
        ];
        ```  
    - Input: None  
    - Output: JSON with current date under path `badges.today`  
    - Edge Cases / Potential Failures:  
      - System time issues (incorrect server time)  
    - Version: Code node v2  

#### 2.4 Data Merging and Appending to Google Sheets

- **Overview:**  
  This block merges the date information with each task record and appends the combined data as rows to a Google Sheet.

- **Nodes Involved:**  
  - Merge2  
  - Daily Progress to Sheet1

- **Node Details:**

  - **Merge2**  
    - Type: Merge node  
    - Configuration:  
      - Mode: Combine (merges data from two inputs into one array)  
      - Combine By: Combine All (creates all combinations of items from both inputs)  
    - Inputs:  
      - Input 0: Today's Date2 (date info)  
      - Input 1: Map Fields (task data)  
    - Output: Merged dataset containing each task enriched with the current date  
    - Edge Cases / Potential Failures:  
      - If either input is empty, output will be empty or incomplete  
    - Version: Merge node v3.2  

  - **Daily Progress to Sheet1**  
    - Type: Google Sheets node  
    - Configuration:  
      - Operation: Append (adds new rows)  
      - Document ID: `1KRiAUbZP77dC_9x5pqrvcQvaAkUsoPXkZOZvfU69ILM` (Google Sheet ID)  
      - Sheet Name/ID: `876214427` (specific sheet tab)  
      - Columns mapped:  
        - `Date` → `badges.today`  
        - `Name` → `Task Name`  
        - `Status` → `Status`  
      - Mapping Mode: Define columns explicitly  
    - Input: Output from Merge2 (merged task + date records)  
    - Output: Confirmation of append operation  
    - Credentials: Google Sheets OAuth2 credentials configured  
    - Edge Cases / Potential Failures:  
      - Invalid or outdated credentials  
      - Incorrect Document ID or Sheet Name causing append to fail  
      - Rate limits or API errors from Google Sheets  
    - Version: Google Sheets node v4.7  

#### 2.5 Setup and Documentation

- **Overview:**  
  Provides detailed setup instructions and resources for configuring Monday.com API credentials, Google Sheets template usage, and credential setup in n8n.

- **Nodes Involved:**  
  - Sticky Note48  
  - Sticky Note14  
  - Sticky Note57  
  - Sticky Note26

- **Node Details (all sticky notes):**

  - **Sticky Note48**  
    - Type: Sticky Note  
    - Content: High-level summary describing the workflow’s purpose to create daily snapshots from Monday.com to Google Sheets.

  - **Sticky Note14**  
    - Type: Sticky Note  
    - Content:  
      - Step-by-step instructions for connecting Monday.com API with link to official docs.  
      - Preparing the Google Sheet template with a provided link.  
      - Instructions for Google Sheets OAuth2 credential setup in n8n.  
      - Contact information for customization help.

  - **Sticky Note57**  
    - Type: Sticky Note  
    - Content:  
      - Specific instructions for generating and configuring the Monday.com Personal API Token.  
      - Link: [Generate Monday API Token](https://developer.monday.com/api-reference/docs/authentication)

  - **Sticky Note26**  
    - Type: Sticky Note  
    - Content:  
      - Instructions for preparing the Google Sheet template with link.  
      - Google Sheets OAuth2 credential setup steps.

- These nodes do not process data but provide critical user guidance and links.

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                         | Input Node(s)          | Output Node(s)     | Sticky Note                                                                                                                                                                                                                   |
|----------------------|---------------------|---------------------------------------|-----------------------|--------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Get many items2       | Monday.com           | Retrieve all tasks from Monday.com board group | None                  | Map Fields         |                                                                                                                                                                                                                               |
| Map Fields           | Set                 | Map and simplify Monday.com task fields | Get many items2       | Merge2             |                                                                                                                                                                                                                               |
| Today's Date2        | Code                | Generate current date for snapshot timestamp | None                  | Merge2             |                                                                                                                                                                                                                               |
| Merge2               | Merge               | Combine date and task data into single dataset | Today's Date2, Map Fields | Daily Progress to Sheet1 |                                                                                                                                                                                                                               |
| Daily Progress to Sheet1 | Google Sheets      | Append daily snapshot rows into Google Sheet | Merge2                 | None               |                                                                                                                                                                                                                               |
| Sticky Note48        | Sticky Note         | Workflow summary and purpose          | None                  | None               | # 📋 Monday.com → Google Sheets Daily Task Status  \n\nThis workflow pulls all tasks from your **Monday.com board** each day and logs them into a **Google Sheet**.  \nIt creates a daily snapshot of your project’s progress and statuses.  |
| Sticky Note14        | Sticky Note         | Setup instructions for Monday.com API and Google Sheets | None                  | None               | ## ⚙️ Setup Instructions  \n\n### 1️⃣ Connect Monday.com API  \n1. In **Monday.com** → go to **Admin → API**  \n2. Copy your **Personal API Token**  \n   - Docs: [Generate Monday API Token](https://developer.monday.com/api-reference/docs/authentication)  \n3. In **n8n → Credentials → New → Monday.com API** → paste your token and save  \n\n---\n\n### 2️⃣ Prepare Your Google Sheet  \n- Copy this template to your own Google Drive: [Google Sheet Template](https://docs.google.com/spreadsheets/d/1KRiAUbZP77dC_9x5pqrvcQvaAkUsoPXkZOZvfU69ILM/edit?gid=876214427#gid=876214427)  \n- Add your data in rows 2–100.  \n- Make sure each new task row starts with `Added = No`.  \n\n#### Connect Google Sheets in n8n  \n1. Go to **n8n → Credentials → New → Google Sheets (OAuth2)**  \n2. Log in with your Google account and grant access  \n3. In the workflow, select your **Spreadsheet ID** and the correct **Sheet Name**  \n\n\n\n## 📬 Contact  \nNeed help customizing this (e.g., filtering by status, emailing daily reports, or adding charts)?  \n\n📧 **robert@ynteractive.com**  \n🔗 **[Robert Breen](https://www.linkedin.com/in/robert-breen-29429625/)**  \n🌐 **[ynteractive.com](https://ynteractive.com)** |
| Sticky Note57        | Sticky Note         | Instructions for Monday.com API token generation | None                  | None               | ### 1️⃣ Connect Monday.com API  \n1. In **Monday.com** → go to **Admin → API**  \n2. Copy your **Personal API Token**  \n   - Docs: [Generate Monday API Token](https://developer.monday.com/api-reference/docs/authentication)  \n3. In **n8n → Credentials → New → Monday.com API** → paste your token and save  \n |
| Sticky Note26        | Sticky Note         | Instructions for Google Sheets setup and OAuth2 credential | None                  | None               | ### 2️⃣ Prepare Your Google Sheet  \n- Copy this template to your own Google Drive: [Google Sheet Template](https://docs.google.com/spreadsheets/d/1KRiAUbZP77dC_9x5pqrvcQvaAkUsoPXkZOZvfU69ILM/edit?gid=876214427#gid=876214427)   \n- Add your data in rows 2–100.  \n- Make sure each new task row starts with `Added = No`.  \n\n#### Connect Google Sheets in n8n  \n1. Go to **n8n → Credentials → New → Google Sheets (OAuth2)**  \n2. Log in with your Google account and grant access.  \n3. In the workflow, select your **Spreadsheet ID** and the correct **Sheet Name**.  \n |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Monday.com API Credentials in n8n:**
   - Go to n8n Settings → Credentials → New Credential → Monday.com API.
   - Obtain your Personal API Token from Monday.com (Admin → API).
   - Paste the token into n8n and save.

2. **Create Google Sheets OAuth2 Credentials:**
   - In n8n, create a new Google Sheets (OAuth2) credential.
   - Authenticate by logging into your Google account.
   - Ensure the Google Sheet you will use is accessible with this account.

3. **Add "Get many items2" Node:**
   - Type: Monday.com node.
   - Operation: `getAll` on `boardItem`.
   - Board ID: Set to your Monday.com board ID (e.g., `9865886546`).
   - Group ID: Set to the target group ID (e.g., `new_group29179`).
   - Credentials: Use the Monday.com API credentials created earlier.

4. **Add "Map Fields" Node:**
   - Type: Set node.
   - Connect input from "Get many items2".
   - Add two fields:  
     - `Task Name` with expression `={{ $json.name }}`  
     - `Status` with expression `={{ $json.column_values[1].text }}`  
   - Purpose: Simplify and select relevant data fields.

5. **Add "Today's Date2" Node:**
   - Type: Code node (JavaScript).
   - No input connections.
   - Code:
     ```js
     return [
       {
         json: {
           badges: {
             today: new Date().toISOString().split('T')[0]
           }
         }
       }
     ];
     ```
   - Purpose: Generate current date for snapshot timestamp.

6. **Add "Merge2" Node:**
   - Type: Merge node.
   - Connect "Today's Date2" output to input 0.
   - Connect "Map Fields" output to input 1.
   - Set mode to `Combine`.
   - Set combine option to `Combine All`.
   - Purpose: Combine date with each task record.

7. **Add "Daily Progress to Sheet1" Node:**
   - Type: Google Sheets node.
   - Connect input from "Merge2".
   - Operation: Append.
   - Set Document ID to your Google Sheet ID (e.g., `1KRiAUbZP77dC_9x5pqrvcQvaAkUsoPXkZOZvfU69ILM`).
   - Set Sheet Name or Sheet ID to the target tab (e.g., `876214427`).
   - Map columns:
     - Date → `badges.today`
     - Name → `Task Name`
     - Status → `Status`
   - Use Google Sheets OAuth2 credentials created earlier.

8. **Add Sticky Notes (Optional but Recommended):**
   - Add sticky notes for:
     - Workflow overview and purpose.
     - Setup instructions for Monday.com API.
     - Setup instructions for Google Sheets template and OAuth2 credential.
   - Include links provided for easy access.

9. **Test the Workflow:**
   - Execute the workflow.
   - Verify that tasks from Monday.com are fetched.
   - Confirm data is correctly mapped and appended with the date into Google Sheets.
   - Check for errors in API calls or data mapping.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                          | Context or Link                                                                                                         |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Monday.com API Token generation and authentication are documented here: [Generate Monday API Token](https://developer.monday.com/api-reference/docs/authentication)                                                                | Monday.com API Token Setup                                                                                              |
| Google Sheet Template used for this workflow can be copied from: [Google Sheet Template](https://docs.google.com/spreadsheets/d/1KRiAUbZP77dC_9x5pqrvcQvaAkUsoPXkZOZvfU69ILM/edit?gid=876214427#gid=876214427)                        | Google Sheets Setup                                                                                                     |
| Contact for customization help: Robert Breen - Email: robert@ynteractive.com, LinkedIn: [Robert Breen](https://www.linkedin.com/in/robert-breen-29429625/), Website: [ynteractive.com](https://ynteractive.com)                      | Support and Customization                                                                                              |
| Ensure that the Monday.com board columns remain consistent, as the workflow assumes the status is in the second column (`column_values[1]`). Changes may require updating the mapping expressions accordingly.                        | Data Mapping Assumptions                                                                                               |
| Be mindful of API rate limits and authentication expiration for both Monday.com and Google Sheets APIs. Schedule workflow runs accordingly and refresh credentials as needed.                                                        | Operational Considerations                                                                                             |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow and complies with applicable content policies. It contains no illegal, offensive, or protected content. All data handled is legal and public.