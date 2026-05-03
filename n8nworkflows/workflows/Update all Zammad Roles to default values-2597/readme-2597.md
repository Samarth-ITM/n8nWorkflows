Update all Zammad Roles to default values

https://n8nworkflows.xyz/workflows/update-all-zammad-roles-to-default-values-2597


# Update all Zammad Roles to default values

### 1. Workflow Overview

This n8n workflow automates the process of resetting all user roles in a Zammad instance to predefined default roles, ensuring consistent role management. It fetches all active users, excludes specified users by ID, and updates the remaining users’ roles to a set of default role IDs.

**Logical Blocks:**

- **1.1 Initialization and Trigger**  
  Entry point via manual trigger and setup of workflow variables including Zammad API credentials, default roles, and exclusion list.

- **1.2 Retrieve System Data**  
  Fetches all existing roles and all active users from Zammad to gather necessary information for updates.

- **1.3 Data Normalization**  
  Transforms raw user and role data into uniform objects for further processing.

- **1.4 Filtering and Decision Making**  
  Filters out excluded users and inactive accounts, deciding which users should have their roles updated.

- **1.5 Role Update Execution**  
  Updates each eligible user’s roles to the defined default roles by making API PUT requests.

- **1.6 Optional Reporting**  
  Converts retrieved roles into an Excel file for audit or review purposes.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization and Trigger

- **Overview:**  
  Starts the workflow manually and sets all required configuration variables such as API credentials, default roles, and excluded users.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’  
  - Basic Variables

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to execute the workflow on demand.  
    - Configuration: No parameters; user manually triggers.  
    - Inputs: None  
    - Outputs: Initiates next node.  
    - Edge cases: None, manual user initiation only.

  - **Basic Variables**  
    - Type: Set  
    - Role: Stores configuration variables such as:  
      - `zammad_base_url`: Zammad instance URL (string)  
      - `zammad_api_key`: API token for authentication (string)  
      - `default_roles`: Array of default role IDs to assign (array of numbers)  
      - `exclude_zammad_users_by_id`: Array of user IDs to exclude (array of numbers)  
    - Configuration: Variables must be filled by user before running.  
    - Inputs: From manual trigger  
    - Outputs: Flows to subsequent API requests.  
    - Edge cases: Missing or incorrect credentials will cause downstream API errors.

---

#### 2.2 Retrieve System Data

- **Overview:**  
  Fetches all roles and all users from the Zammad API using provided credentials.

- **Nodes Involved:**  
  - Get all Roles  
  - Get all Users

- **Node Details:**

  - **Get all Roles**  
    - Type: HTTP Request  
    - Role: Retrieves all roles defined in Zammad via GET `/api/v1/roles`.  
    - Configuration:  
      - URL built dynamically using `zammad_base_url` variable.  
      - Auth via Bearer token in `Authorization` header (`zammad_api_key`).  
      - Returns all roles (no filter).  
    - Inputs: From Basic Variables  
    - Outputs: JSON array of roles.  
    - Edge cases: Auth failure, network timeout, empty role list.

  - **Get all Users**  
    - Type: Zammad node (custom integration)  
    - Role: Retrieves all users from Zammad with `returnAll=true`.  
    - Configuration: Default filters (empty).  
    - Auth: Uses Zammad Token Auth credentials set in n8n.  
    - Inputs: From Basic Variables  
    - Outputs: JSON array of all users.  
    - Edge cases: Auth errors, large user base causing timeout or rate limits.

---

#### 2.3 Data Normalization

- **Overview:**  
  Converts raw API data into standardized user and role objects for consistent processing in the workflow.

- **Nodes Involved:**  
  - Zammad Univeral Role Object  
  - Zammad Univeral User Object

- **Node Details:**

  - **Zammad Univeral Role Object**  
    - Type: Set  
    - Role: Extracts and retains only relevant role data fields:  
      - `role_id` (number)  
      - `name` (string)  
    - Configuration: Uses expressions to map fields from input JSON.  
    - Inputs: From Get all Roles node  
    - Outputs: Cleaned role data for filtering or export.  
    - Edge cases: Missing fields in API response.

  - **Zammad Univeral User Object**  
    - Type: Set  
    - Role: Extracts and formats user fields:  
      - `user_id` (number)  
      - `organization_id` (number)  
      - `email` (string)  
      - `firstname` (string)  
      - `lastname` (string)  
      - `role_ids` (string, comma-separated from array)  
      - `groups` (array from group IDs)  
      - `active` (boolean)  
    - Configuration: Uses expressions with `$json` to map fields, joins arrays where needed.  
    - Inputs: From Get all Users node  
    - Outputs: Uniform user objects for filtering.  
    - Edge cases: Handling users with missing or malformed role/group data.

---

#### 2.4 Filtering and Decision Making

- **Overview:**  
  Filters out excluded users and inactive users before proceeding to update roles.

- **Nodes Involved:**  
  - If (Conditional Node)  
  - Filter Roles if needed (present but optional/filtering roles not fully used)

- **Node Details:**

  - **If**  
    - Type: If (Condition)  
    - Role: Decides whether to update a user by checking:  
      - User ID not in `exclude_zammad_users_by_id` array (using `notContains`)  
      - User is active (`active` boolean true)  
      - One unused condition with empty values (likely placeholder / no effect)  
    - Configuration: All conditions combined with AND.  
    - Inputs: From Zammad Univeral User Object  
    - Outputs: True branch proceeds to update; false branch skips.  
    - Edge cases: Empty exclusion list means no users excluded; inactive users skipped.

  - **Filter Roles if needed**  
    - Type: If (Conditional Node)  
    - Role: Present but effectively a placeholder; checks if JSON exists.  
    - Inputs: From Zammad Univeral Role Object  
    - Outputs: Connects to Excel export; no direct impact on update logic.  
    - Edge cases: No filtering applied, safe to omit.

---

#### 2.5 Role Update Execution

- **Overview:**  
  Performs the actual update of each eligible user's roles by sending PUT requests to the Zammad API.

- **Nodes Involved:**  
  - Update Users to default Role(s)

- **Node Details:**

  - **Update Users to default Role(s)**  
    - Type: HTTP Request  
    - Role: Updates a single user's roles via PUT `/api/v1/users/{user_id}` with JSON body:  
      ```json
      {
        "role_ids": [<default_roles array>]
      }
      ```  
    - Configuration:  
      - URL dynamically built using `zammad_base_url` and current user's `user_id`.  
      - Auth header uses Bearer token from `zammad_api_key`.  
      - On error, continues execution (does not stop workflow).  
    - Inputs: From If node’s true branch (filtered users)  
    - Outputs: HTTP response for each update; no further downstream nodes.  
    - Edge cases:  
      - API errors (e.g., permissions, invalid user ID)  
      - Network issues  
      - Partial failures handled by `continueErrorOutput`.  
    - Version-specific: Uses HTTP Request node version 4.2.

---

#### 2.6 Optional Reporting

- **Overview:**  
  Converts the full list of roles into an Excel file for reporting or review.

- **Nodes Involved:**  
  - Convert to Excel Roles

- **Node Details:**

  - **Convert to Excel Roles**  
    - Type: Convert To File  
    - Role: Converts JSON role data to `.xlsx` file named "Zammad_Roles.xlsx".  
    - Configuration:  
      - Operation: XLSX  
      - Filename configured as "Zammad_Roles.xlsx".  
    - Inputs: From Filter Roles if needed node  
    - Outputs: Binary XLSX file (can be used downstream or saved externally).  
    - Edge cases: Empty role list results in empty Excel file.

---

### 3. Summary Table

| Node Name                   | Node Type          | Functional Role                                | Input Node(s)                 | Output Node(s)               | Sticky Note                                     |
|-----------------------------|--------------------|-----------------------------------------------|------------------------------|-----------------------------|------------------------------------------------|
| When clicking ‘Test workflow’| Manual Trigger     | Entry point to start the workflow manually    | None                         | Basic Variables              |                                                |
| Basic Variables             | Set                | Stores API credentials and configuration vars | When clicking ‘Test workflow’ | Get all Roles, Get all Users |                                                |
| Get all Roles               | HTTP Request       | Fetches all roles from Zammad API              | Basic Variables              | Zammad Univeral Role Object  |                                                |
| Zammad Univeral Role Object | Set                | Normalize roles into simplified objects        | Get all Roles                | Filter Roles if needed       |                                                |
| Filter Roles if needed      | If                 | Placeholder filter on roles, no effect         | Zammad Univeral Role Object  | Convert to Excel Roles       |                                                |
| Convert to Excel Roles      | Convert To File    | Converts roles JSON to Excel file               | Filter Roles if needed       | None                        |                                                |
| Get all Users               | Zammad             | Fetch all users from Zammad                     | Basic Variables              | Zammad Univeral User Object  |                                                |
| Zammad Univeral User Object | Set                | Normalize user data into uniform objects        | Get all Users                | If                          |                                                |
| If                         | If                 | Filters users by exclusion list and active flag| Zammad Univeral User Object  | Update Users to default Role(s) |                                                |
| Update Users to default Role(s) | HTTP Request   | Updates user roles to default via API PUT       | If                          | None                        | Set node to continue on error to avoid workflow stop |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node**  
   - Name: `When clicking ‘Test workflow’`  
   - No parameters.

2. **Add a Set node for basic variables**  
   - Name: `Basic Variables`  
   - Configure variables:  
     - `zammad_base_url` (string): Your Zammad instance URL  
     - `zammad_api_key` (string): Your API token  
     - `default_roles` (array of numbers): Default role IDs to assign  
     - `exclude_zammad_users_by_id` (array of numbers): User IDs to exclude from update  
   - Connect `When clicking ‘Test workflow’` → `Basic Variables`

3. **Add HTTP Request node to get all roles**  
   - Name: `Get all Roles`  
   - Method: GET  
   - URL: `={{ $json.zammad_base_url }}/api/v1/roles`  
   - Headers: `Authorization: Bearer {{ $json.zammad_api_key }}`  
   - Connect `Basic Variables` → `Get all Roles`

4. **Add Zammad node to get all users**  
   - Name: `Get all Users`  
   - Operation: Get All  
   - Return All: true  
   - Credentials: Use Zammad Token Auth with your token  
   - Connect `Basic Variables` → `Get all Users`

5. **Add Set node to normalize roles**  
   - Name: `Zammad Univeral Role Object`  
   - Fields:  
     - `role_id`: `={{ $json.id }}` (number)  
     - `name`: `={{ $json.name }}` (string)  
   - Connect `Get all Roles` → `Zammad Univeral Role Object`

6. **Add Set node to normalize users**  
   - Name: `Zammad Univeral User Object`  
   - Fields:  
     - `user_id`: `={{ $json.id }}` (number)  
     - `organization_id`: `={{ $json.organization_id }}` (number)  
     - `email`: `={{ $json.email }}` (string)  
     - `firstname`: `={{ $json.firstname }}` (string)  
     - `lastname`: `={{ $json.lastname }}` (string)  
     - `role_ids`: `={{ $json.role_ids.join() }}` (string)  
     - `groups`: `={{ $json.group_ids }}` (array)  
     - `active`: `={{ $json.active }}` (boolean)  
   - Connect `Get all Users` → `Zammad Univeral User Object`

7. **Add an If node for filtering users**  
   - Name: `If`  
   - Conditions (AND):  
     - Expression: Check `exclude_zammad_users_by_id` array from Basic Variables does NOT contain current `user_id`  
       `={{ $('Basic Variables').item.json.exclude_zammad_users_by_id}}` NOT CONTAINS `{{$json.user_id}}`  
     - `active` is true: `={{ $json.active }}` equals true  
   - Connect `Zammad Univeral User Object` → `If`

8. **Add HTTP Request node to update user roles**  
   - Name: `Update Users to default Role(s)`  
   - Method: PUT  
   - URL: `={{ $('Basic Variables').item.json.zammad_base_url }}/api/v1/users/{{ $json.user_id }}`  
   - Headers: `Authorization: Bearer {{ $('Basic Variables').item.json.zammad_api_key }}`  
   - Body Type: JSON  
   - Body:  
     ```json
     {
       "role_ids": [ {{ $('Basic Variables').item.json.default_roles }} ]
     }
     ```  
   - On Error: Continue on error output (to avoid full workflow failure)  
   - Connect True output of `If` → `Update Users to default Role(s)`

9. **Optional: Add If node to filter roles and Convert to File node for Excel export**  
   - Name: `Filter Roles if needed` (If node)  
     - Condition: Check if role JSON exists (optional, can be omitted)  
   - Name: `Convert to Excel Roles`  
     - Operation: XLSX  
     - Filename: `Zammad_Roles.xlsx`  
   - Connect `Zammad Univeral Role Object` → `Filter Roles if needed` → `Convert to Excel Roles`

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| For issues or suggestions, visit the GitHub repository: https://github.com/Sirhexalot/n8n-Update-all-Zammad-Roles-to-default-values | Project Repository and Support |

---

This completes the detailed analysis and documentation of the "Update all Zammad Roles to default values" n8n workflow, providing a comprehensive reference for understanding, modification, and re-creation.