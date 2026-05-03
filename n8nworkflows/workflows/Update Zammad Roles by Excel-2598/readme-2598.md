Update Zammad Roles by Excel

https://n8nworkflows.xyz/workflows/update-zammad-roles-by-excel-2598


# Update Zammad Roles by Excel

### 1. Workflow Overview

This n8n workflow automates updating user roles in a Zammad helpdesk system based on user data provided in an Excel file. It is designed to synchronize role assignments efficiently by matching users via email addresses. The workflow is particularly suited for organizations managing user roles centrally via spreadsheets and needing to reflect changes quickly in Zammad.

**Logical Blocks:**

- **1.1 Input Initialization and Variable Setup:** Defines essential variables and triggers the workflow.
- **1.2 Excel File Download and Data Extraction:** Downloads the Excel file from a specified URL and extracts the user data.
- **1.3 Data Transformation:** Prepares and formats data, focusing on extracting emails and role IDs.
- **1.4 Zammad User Lookup:** Queries Zammad API to find existing users by email.
- **1.5 Data Merge:** Combines Excel data with Zammad user data to align records.
- **1.6 Role Update Execution:** Sends API requests to update user roles in Zammad, handling errors gracefully.
- **1.7 Authentication Setup:** Provides guidance notes on configuring API authentication headers.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Variable Setup

- **Overview:**  
  This block starts the workflow manually and sets global variables for the Zammad instance URL and the Excel data source URL.

- **Nodes Involved:**  
  - When clicking "Execute Workflow" (Manual Trigger)  
  - Basic Variables (Set)

- **Node Details:**  

  - **When clicking "Execute Workflow"**  
    - Type: Manual Trigger  
    - Role: Initiates workflow execution manually.  
    - Configuration: Default, no parameters.  
    - Inputs: None  
    - Outputs: Starts next node on trigger.  
    - Edge Cases: No major failure modes; user must manually trigger.

  - **Basic Variables**  
    - Type: Set  
    - Role: Assigns key string variables for workflow configuration.  
    - Configuration:  
      - `zammad_base_url`: URL of the Zammad instance (e.g., https://zammad.sirhexalot.de/)  
      - `excel_source_url`: URL of the Excel file containing user data (e.g., http://zammad.sirhexalot.de/Users.txt)  
    - Expressions used: Hardcoded string values.  
    - Inputs: From Manual Trigger  
    - Outputs: Passes variables forward.  
    - Edge Cases: Variables must be correct URLs; otherwise, downstream HTTP requests will fail (e.g., 404 or connection error).

---

#### 2.2 Excel File Download and Data Extraction

- **Overview:**  
  Downloads the Excel file from the URL and extracts its contents to JSON for processing.

- **Nodes Involved:**  
  - Download Excel (HTTP Request)  
  - Extract from File (Extract from File)

- **Node Details:**  

  - **Download Excel**  
    - Type: HTTP Request  
    - Role: Downloads the Excel file as a binary file from the provided URL.  
    - Configuration:  
      - URL: `={{ $json.excel_source_url }}` (dynamic from previous variable)  
      - Response Format: File (binary)  
    - Inputs: Receives variables from Basic Variables node.  
    - Outputs: Binary file data (Excel spreadsheet) to next node.  
    - Edge Cases:  
      - HTTP errors (404, 403, timeout) if URL is invalid or inaccessible.  
      - Large file size could cause delays or memory issues.

  - **Extract from File**  
    - Type: Extract from File  
    - Role: Parses the downloaded Excel file into JSON rows.  
    - Configuration:  
      - Operation: 'xlsx' (Excel format)  
      - Default options for reading.  
    - Inputs: Binary Excel file from Download Excel node.  
    - Outputs: JSON array of user data objects (including emails and role assignments).  
    - Edge Cases:  
      - Malformed or corrupted Excel files cause parsing errors.  
      - Empty or unexpected sheet structure could result in missing data.

---

#### 2.3 Data Transformation

- **Overview:**  
  Reformats the extracted JSON data, focusing on isolating the email and role_ids fields for further processing.

- **Nodes Involved:**  
  - Zammad Universal User Object (Set)

- **Node Details:**  

  - **Zammad Universal User Object**  
    - Type: Set  
    - Role: Creates a simplified JSON object containing only the email and role_ids fields for each user entry.  
    - Configuration:  
      - Keeps only two fields:  
        - `email`: from extracted JSON `$json.email`  
        - `role_ids`: from extracted JSON `$json.role_ids` (with a trailing newline which might be unintended)  
      - `keepOnlySet`: true (removes other fields)  
    - Inputs: JSON rows from Extract from File node.  
    - Outputs: Cleaned user objects for merging and API queries.  
    - Edge Cases:  
      - Missing `email` or `role_ids` fields in source data may cause empty or invalid user objects.  
      - Trailing newline in `role_ids` could introduce formatting issues.

---

#### 2.4 Zammad User Lookup

- **Overview:**  
  Searches the Zammad system for users matching each email from the Excel data to retrieve user IDs and other metadata.

- **Nodes Involved:**  
  - Find Zammad User by email (HTTP Request)

- **Node Details:**  

  - **Find Zammad User by email**  
    - Type: HTTP Request  
    - Role: Queries Zammad API endpoint `/api/v1/users/search` filtering by email.  
    - Configuration:  
      - URL template: `{{ $('Basic Variables').item.json.zammad_base_url }}api/v1/users/search?query=email:{{ $json.email }}`  
      - Authentication: HTTP Header Authentication with `Authorization: Bearer <token>`  
      - Method: GET (default)  
      - `executeOnce`: false (runs per item)  
      - `alwaysOutputData`: false (outputs only on success)  
    - Inputs: Clean user objects from Zammad Universal User Object node.  
    - Outputs: Zammad user data, including user `id` to be used for updates.  
    - Edge Cases:  
      - API errors such as 401 Unauthorized if token is invalid, 404 if endpoint incorrect.  
      - Users not found: results may be empty, causing downstream merge issues.  
      - Rate limiting by API could cause throttling or failures.

---

#### 2.5 Data Merge

- **Overview:**  
  Combines the original Excel user data with the Zammad user data by matching on the email field, preparing for role updates.

- **Nodes Involved:**  
  - Merge (Merge)

- **Node Details:**  

  - **Merge**  
    - Type: Merge  
    - Role: Combines two data streams based on the `email` field to align Excel data with corresponding Zammad user info.  
    - Configuration:  
      - Mode: Combine  
      - Fields to match: `email` (string)  
    - Inputs:  
      - Left input: Output of Find Zammad User by email (Zammad user data)  
      - Right input: Output of Zammad Universal User Object (Excel user data)  
    - Outputs: Merged user objects containing Zammad user ID and desired role_ids.  
    - Edge Cases:  
      - Emails existing in Excel but missing in Zammad cause incomplete merges.  
      - Duplicate emails in either source may cause multiple matches and unexpected behavior.

---

#### 2.6 Role Update Execution

- **Overview:**  
  Updates user roles in Zammad by sending PUT requests to the API for each matched user ID with new role IDs.

- **Nodes Involved:**  
  - Update User Roles (HTTP Request)

- **Node Details:**  

  - **Update User Roles**  
    - Type: HTTP Request  
    - Role: Sends PUT request to Zammad API to update the `role_ids` of a user identified by `id`.  
    - Configuration:  
      - URL: `{{ $('Basic Variables').item.json.zammad_base_url }}/api/v1/users/{{ $json.id }}`  
      - Method: PUT  
      - Body Type: JSON  
      - JSON Body:  
        ```json
        {
          "role_ids": [ {{ $json.role_ids }} ]
        }
        ```  
        (Note: `role_ids` must be formatted properly as an array of IDs.)  
      - Authentication: HTTP Header Authentication with bearer token.  
      - On Error: Continue (does not halt workflow on failure)  
    - Inputs: Merged user data from Merge node.  
    - Outputs: API response from update attempt.  
    - Edge Cases:  
      - Invalid `role_ids` format or empty array may cause API errors.  
      - Network or API errors (401, 403, 404, 500).  
      - User IDs not found or already deleted cause failures.  
      - Partial updates may lead to inconsistent states.  
      - Errors are logged but workflow continues.

---

#### 2.7 Authentication Setup (Sticky Note)

- **Overview:**  
  Provides instructions on how to configure authentication credentials for the HTTP requests to Zammad’s API.

- **Nodes Involved:**  
  - Sticky Note

- **Node Details:**  

  - **Sticky Note**  
    - Type: Sticky Note  
    - Content:  
      ```
      ## Authentication for Zammad
      
      Create in the Node Find Zammad User by email a Header Auth Authentication
      
      Use:
      
      Name: Authorization
      Value: Bearer - put here your zammad api token -
      ```  
    - Role: Documentation for users to set up HTTP Header Auth credentials properly.  
    - Position: Separate visual note, no data input/output.  
    - Edge Cases: Missing or incorrect token results in authentication failure.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                         | Input Node(s)                 | Output Node(s)                     | Sticky Note                                                                                       |
|----------------------------|---------------------|---------------------------------------|------------------------------|----------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger      | Starts workflow execution               | None                         | Basic Variables                   |                                                                                                 |
| Basic Variables            | Set                 | Sets config variables (Zammad URL, Excel URL) | When clicking "Execute Workflow" | Download Excel                   |                                                                                                 |
| Download Excel             | HTTP Request        | Downloads Excel file from URL           | Basic Variables               | Extract from File                 |                                                                                                 |
| Extract from File          | Extract from File   | Parses Excel file to JSON                | Download Excel                | Zammad Universal User Object      |                                                                                                 |
| Zammad Universal User Object | Set                 | Extracts email and role_ids fields from data | Extract from File             | Merge, Find Zammad User by email  |                                                                                                 |
| Find Zammad User by email  | HTTP Request        | Looks up users in Zammad by email       | Zammad Universal User Object  | Merge                           | Create in the Node Find Zammad User by email a Header Auth Authentication with: Name: Authorization, Value: Bearer <token> |
| Merge                     | Merge               | Joins Excel data with Zammad user data  | Find Zammad User by email, Zammad Universal User Object | Update User Roles               |                                                                                                 |
| Update User Roles          | HTTP Request        | Updates user roles in Zammad             | Merge                        | None                            |                                                                                                 |
| Sticky Note               | Sticky Note         | Instructions for authentication setup   | None                         | None                            | Create in the Node Find Zammad User by email a Header Auth Authentication Name: Authorization Value: Bearer - put here your zammad api token - |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add node of type **Manual Trigger** named "When clicking \"Execute Workflow\"".  
   - No parameters needed.

2. **Set Basic Variables:**  
   - Add a **Set** node named "Basic Variables".  
   - Create two string variables:  
     - `zammad_base_url` with your Zammad instance URL (e.g., `https://zammad.sirhexalot.de/`).  
     - `excel_source_url` with the HTTP URL of your Excel file containing user emails and roles (e.g., `http://zammad.sirhexalot.de/Users.txt`).  
   - Connect "When clicking \"Execute Workflow\"" → "Basic Variables".

3. **Download Excel File:**  
   - Add an **HTTP Request** node named "Download Excel".  
   - Set URL to expression: `={{ $json.excel_source_url }}`.  
   - Under options, set Response Format to `File` (to download the file as binary).  
   - Connect "Basic Variables" → "Download Excel".

4. **Extract Excel Content:**  
   - Add an **Extract from File** node named "Extract from File".  
   - Set operation to `xlsx`.  
   - Connect "Download Excel" → "Extract from File".

5. **Prepare User Data:**  
   - Add a **Set** node named "Zammad Universal User Object".  
   - Configure to keep only two string fields:  
     - `email` with expression `={{ $json.email }}`  
     - `role_ids` with expression `={{ $json.role_ids }}` (ensure no trailing whitespace in your source data).  
   - Enable "Keep Only Set".  
   - Connect "Extract from File" → "Zammad Universal User Object".

6. **Configure HTTP Header Authentication Credential:**  
   - Go to Credentials in n8n.  
   - Create a new **HTTP Header Auth** credential for Zammad API with:  
     - Name: `Authorization`  
     - Value: `Bearer <your_zammad_api_token>` (replace with your actual token).  

7. **Find Zammad User by Email:**  
   - Add an **HTTP Request** node named "Find Zammad User by email".  
   - Set Request URL to expression:  
     `={{ $('Basic Variables').item.json.zammad_base_url }}api/v1/users/search?query=email:{{ $json.email }}`  
   - Use HTTP Header Auth credentials created above.  
   - Method: GET (default).  
   - Set "Execute Once" to false (default) to run per item.  
   - Connect "Zammad Universal User Object" → "Find Zammad User by email".

8. **Merge Data Streams:**  
   - Add a **Merge** node named "Merge".  
   - Set mode to `Combine`.  
   - Set "Fields to Match" to `email` (string).  
   - Connect:  
     - Left input from "Find Zammad User by email".  
     - Right input from "Zammad Universal User Object".

9. **Update User Roles in Zammad:**  
   - Add an **HTTP Request** node named "Update User Roles".  
   - Set URL to expression:  
     `={{ $('Basic Variables').item.json.zammad_base_url }}/api/v1/users/{{ $json.id }}`  
   - Set method to PUT.  
   - Set Body Content Type to JSON.  
   - Set JSON Body to:  
     ```json
     {
       "role_ids": [ {{ $json.role_ids }} ]
     }
     ```  
   - Use the same HTTP Header Auth credential as before.  
   - Set on-error behavior to "Continue" to avoid halting if one update fails.  
   - Connect "Merge" → "Update User Roles".

10. **Optional: Add Sticky Note for Documentation**  
    - Add a **Sticky Note** node with instructions on setting up the HTTP Header Authentication for Zammad API calls, including the name and value format.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                      |
|-------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| For issues or suggestions, visit the GitHub Repository: https://github.com/Sirhexalot/Update-n8n-Zammad-Roles-by-Excel | Issue tracking and workflow improvements                            |
| Authentication for Zammad API requires HTTP Header Auth with `Authorization: Bearer <token>`                      | Refer to Sticky Note in workflow and credential setup instructions |
| Ensure the Excel file's `role_ids` column properly formats role IDs as a comma-separated list without trailing spaces or newlines | Avoid parsing and API errors during role updates                   |

---

This reference document fully describes the "Update Roles by Excel" workflow, enabling users and automation agents to understand, reproduce, and maintain the workflow effectively.