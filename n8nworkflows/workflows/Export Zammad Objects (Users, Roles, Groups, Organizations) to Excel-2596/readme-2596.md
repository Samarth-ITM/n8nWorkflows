Export Zammad Objects (Users, Roles, Groups, Organizations) to Excel

https://n8nworkflows.xyz/workflows/export-zammad-objects--users--roles--groups--organizations--to-excel-2596


# Export Zammad Objects (Users, Roles, Groups, Organizations) to Excel

### 1. Workflow Overview

This n8n workflow automates the export of key Zammad data objects—Users, Roles, Groups, and Organizations—into separate Excel files. It is designed for administrators or data analysts who need to extract, consolidate, and report on Zammad system data efficiently.

The workflow consists of these logical blocks:

- **1.1 Input Trigger and Initialization**  
  Starts execution manually and sets base variables including Zammad API credentials.

- **1.2 Data Retrieval from Zammad API**  
  Fetches all Users, Roles, Groups, and Organizations from the Zammad system using API calls.

- **1.3 Data Standardization**  
  Transforms raw Zammad data into unified, simplified objects for Users, Roles, Groups, and Organizations.

- **1.4 Optional Data Filtering**  
  Applies filters to each data category if needed (currently configured but inactive).

- **1.5 Export to Excel Files**  
  Converts the standardized data sets into separate Excel (.xlsx) files for Users, Roles, Groups, and Organizations.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Trigger and Initialization

**Overview:**  
This block initiates the workflow manually and sets essential variables such as the Zammad base URL and API key.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Basic Variables

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Entry point; starts workflow on manual user action  
  - Configuration: Default manual trigger (no parameters)  
  - Input: None  
  - Output: Triggers downstream nodes  
  - Edge Cases: None likely; manual start

- **Basic Variables**  
  - Type: Set  
  - Role: Stores Zammad API credentials as workflow variables  
  - Configuration:  
    - `zammad_base_url`: Placeholder string for the Zammad instance URL  
    - `zammad_api_key`: Placeholder string for API key  
  - Input: Triggered by manual trigger node  
  - Output: Supplies credentials to API request nodes  
  - Edge Cases: Missing or incorrect credentials will cause authentication failures downstream

---

#### 2.2 Data Retrieval from Zammad API

**Overview:**  
Fetches complete lists of Users, Roles, Groups, and Organizations from Zammad using HTTP requests or native Zammad nodes.

**Nodes Involved:**  
- Get all Users  
- Get all Organizations  
- Get all Roles  
- Get all Groups

**Node Details:**

- **Get all Users**  
  - Type: Zammad node (resource: user)  
  - Role: Retrieves all user records with full details  
  - Configuration:  
    - Operation: getAll  
    - Return All: true  
    - Credentials: Zammad Token Auth (API key)  
  - Input: Triggered by manual trigger node  
  - Output: Array of user objects  
  - Edge Cases: API errors, rate limits, empty user list

- **Get all Organizations**  
  - Type: Zammad node (resource: organization)  
  - Role: Retrieves all organizations  
  - Configuration: Same as Users node but for organizations  
  - Input: Triggered by manual trigger node  
  - Output: Array of organization objects  
  - Edge Cases: API errors, empty organization list

- **Get all Roles**  
  - Type: HTTP Request  
  - Role: Fetches all roles via REST API call  
  - Configuration:  
    - URL: `{{ $json.zammad_base_url }}/api/v1/roles`  
    - Method: GET  
    - Headers: Authorization Bearer token from `zammad_api_key` variable  
  - Input: Receives credentials from Basic Variables node  
  - Output: List of roles in JSON format  
  - Edge Cases: Invalid URL or token, HTTP errors, empty roles list

- **Get all Groups**  
  - Type: HTTP Request  
  - Role: Fetches all groups via REST API call  
  - Configuration: Similar to Get all Roles but URL ends with `/groups`  
  - Input: From Basic Variables node for credentials  
  - Output: List of groups in JSON  
  - Edge Cases: Same as Get all Roles

---

#### 2.3 Data Standardization

**Overview:**  
Transforms raw data from Zammad into uniform, simplified objects to facilitate export and future processing.

**Nodes Involved:**  
- Zammad Univeral User Object  
- Zammad Univeral Organization Object  
- Zammad Univeral Role Object  
- Zammad Univeral Group Object

**Node Details:**

- **Zammad Univeral User Object**  
  - Type: Set  
  - Role: Maps user attributes to simplified object with fields: user_id, organization_id, email, firstname, lastname, role_ids (comma-separated), groups (array)  
  - Key Expressions: Uses expressions like `{{$json.role_ids.join()}}` to convert arrays to strings  
  - Input: From Get all Users node  
  - Output: Standardized user objects  
  - Edge Cases: Missing fields in original data, empty arrays causing join to return empty string

- **Zammad Univeral Organization Object**  
  - Type: Set  
  - Role: Maps organization id and name  
  - Input: From Get all Organizations node  
  - Output: Standardized organization objects  
  - Edge Cases: Missing name or id fields

- **Zammad Univeral Role Object**  
  - Type: Set  
  - Role: Extracts role id and name  
  - Input: From Get all Roles node  
  - Output: Simplified role objects  
  - Edge Cases: Missing role name or id

- **Zammad Univeral Group Object**  
  - Type: Set  
  - Role: Extracts group id and name  
  - Input: From Get all Groups node  
  - Output: Simplified group objects  
  - Edge Cases: Missing group name or id

---

#### 2.4 Optional Data Filtering

**Overview:**  
Provides conditional filtering on the standardized data objects but is currently inactive (filters check for specific conditions not met by default data).

**Nodes Involved:**  
- If (for Users)  
- Filter Organizations if needed  
- Filter Roles if needed  
- Filter Groups if needed

**Node Details:**

- Each filter node is an If node checking for presence or specific values in the data.  
- Filters check if the current item exists or matches a fixed value (e.g., 1781).  
- Input: Standardized objects nodes for each data type  
- Output: Either passes data downstream for Excel conversion or blocks it  
- Edge Cases: Filters are mostly placeholders; misconfiguration would block all data output

---

#### 2.5 Export to Excel Files

**Overview:**  
Converts each filtered or unfiltered data set into an Excel file named accordingly.

**Nodes Involved:**  
- Convert to Excel Users  
- Convert to Excel Organizations  
- Convert to Excel Roles  
- Convert to Excel Groups

**Node Details:**

- All nodes are ConvertToFile type with operation set to `xlsx`.  
- File names set explicitly:  
  - Users: `Zammad_Users.xslx` (note possible typo in extension)  
  - Organizations: `Zammad_Organizations.xlsx`  
  - Roles: `Zammad_Roles.xlsx`  
  - Groups: `Zammad_Groups.xlsx`  
- Input: From respective filter nodes or directly from standardized nodes if filters are inactive  
- Output: Excel file data ready for download or further processing  
- Edge Cases: Conversion errors if input data is empty or malformed, typo in Users file extension may cause file recognition issues

---

### 3. Summary Table

| Node Name                    | Node Type          | Functional Role                          | Input Node(s)                  | Output Node(s)                      | Sticky Note                                 |
|------------------------------|--------------------|----------------------------------------|-------------------------------|------------------------------------|---------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Starts workflow manually                | None                          | Get all Users, Basic Variables, Get all Organizations |                                             |
| Basic Variables              | Set                | Sets Zammad API credentials             | When clicking ‘Test workflow’ | Get all Roles, Get all Groups       |                                             |
| Get all Users                | Zammad             | Retrieves all Users                     | When clicking ‘Test workflow’ | Zammad Univeral User Object         |                                             |
| Get all Organizations        | Zammad             | Retrieves all Organizations             | When clicking ‘Test workflow’ | Zammad Univeral Organization Object |                                             |
| Get all Roles                | HTTP Request       | Retrieves all Roles                     | Basic Variables               | Zammad Univeral Role Object         |                                             |
| Get all Groups               | HTTP Request       | Retrieves all Groups                    | Basic Variables               | Zammad Univeral Group Object        |                                             |
| Zammad Univeral User Object  | Set                | Standardizes User data                  | Get all Users                 | If                                |                                             |
| Zammad Univeral Organization Object | Set          | Standardizes Organization data          | Get all Organizations         | Filter Organizations if needed      |                                             |
| Zammad Univeral Role Object  | Set                | Standardizes Role data                  | Get all Roles                 | Filter Roles if needed              |                                             |
| Zammad Univeral Group Object | Set                | Standardizes Group data                 | Get all Groups                | Filter Groups if needed             |                                             |
| If                          | If                 | Optional user data filtering            | Zammad Univeral User Object   | Convert to Excel Users              |                                             |
| Filter Organizations if needed | If               | Optional organization filtering         | Zammad Univeral Organization Object | Convert to Excel Organizations |                                             |
| Filter Roles if needed       | If                 | Optional role filtering                 | Zammad Univeral Role Object   | Convert to Excel Roles              |                                             |
| Filter Groups if needed      | If                 | Optional group filtering                | Zammad Univeral Group Object  | Convert to Excel Groups             |                                             |
| Convert to Excel Users       | ConvertToFile      | Converts Users data to Excel file      | If                           | None                              | Filename extension typo: "Zammad_Users.xslx" |
| Convert to Excel Organizations | ConvertToFile    | Converts Organizations data to Excel   | Filter Organizations if needed | None                              |                                             |
| Convert to Excel Roles       | ConvertToFile      | Converts Roles data to Excel            | Filter Roles if needed        | None                              |                                             |
| Convert to Excel Groups      | ConvertToFile      | Converts Groups data to Excel           | Filter Groups if needed       | None                              |                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Test workflow’`  
   - Type: Manual Trigger  
   - No additional configuration needed

2. **Create Set Node for Basic Variables**  
   - Name: `Basic Variables`  
   - Type: Set  
   - Add two string variables:  
     - `zammad_base_url` with placeholder `-put-your-zammad-base-url-`  
     - `zammad_api_key` with placeholder `-put-your-api-key-`  
   - Connect output of manual trigger to this node

3. **Create Zammad Node to Get All Users**  
   - Name: `Get all Users`  
   - Type: Zammad (resource: user, operation: getAll)  
   - Set `Return All` to true  
   - Configure credentials with your Zammad Token Auth API key  
   - Connect manual trigger output to this node

4. **Create Zammad Node to Get All Organizations**  
   - Name: `Get all Organizations`  
   - Type: Zammad (resource: organization, operation: getAll)  
   - Set `Return All` to true  
   - Use same credentials as Users node  
   - Connect manual trigger output to this node

5. **Create HTTP Request Node to Get All Roles**  
   - Name: `Get all Roles`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `={{ $json.zammad_base_url }}/api/v1/roles`  
   - Add header: `Authorization: Bearer {{ $json.zammad_api_key }}`  
   - Connect Basic Variables node output to this node

6. **Create HTTP Request Node to Get All Groups**  
   - Name: `Get all Groups`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `={{ $json.zammad_base_url }}/api/v1/groups`  
   - Add header: `Authorization: Bearer {{ $json.zammad_api_key }}`  
   - Connect Basic Variables node output to this node

7. **Create Set Node to Standardize User Object**  
   - Name: `Zammad Univeral User Object`  
   - Type: Set  
   - Keep only set fields  
   - Map fields:  
     - `user_id` = `{{$json.id}}`  
     - `organization_id` = `{{$json.organization_id}}`  
     - `email` = `{{$json.email}}`  
     - `firstname` = `{{$json.firstname}}`  
     - `lastname` = `{{$json.lastname}}`  
     - `role_ids` = `{{$json.role_ids.join()}}` (comma-separated string)  
     - `groups` = `{{$json.group_ids}}` (array)  
   - Connect output of Get all Users node here

8. **Create Set Node to Standardize Organization Object**  
   - Name: `Zammad Univeral Organization Object`  
   - Type: Set  
   - Keep only set fields  
   - Map fields:  
     - `organization_id` = `{{$json.id}}`  
     - `name` = `{{$json.name}}`  
   - Connect output of Get all Organizations node here

9. **Create Set Node to Standardize Role Object**  
   - Name: `Zammad Univeral Role Object`  
   - Type: Set  
   - Keep only set fields  
   - Map fields:  
     - `role_id` = `{{$json.id}}`  
     - `name` = `{{$json.name}}`  
   - Connect output of Get all Roles node here

10. **Create Set Node to Standardize Group Object**  
    - Name: `Zammad Univeral Group Object`  
    - Type: Set  
    - Keep only set fields  
    - Map fields:  
      - `group_id` = `{{$json.id}}`  
      - `name` = `{{$json.name}}`  
    - Connect output of Get all Groups node here

11. **(Optional) Create If Nodes for Filtering**  
    - For each data type (Users, Organizations, Roles, Groups), create an `If` node to filter data if needed (currently optional and configured to pass all data)  
    - Connect standardized data nodes to their respective filters

12. **Create ConvertToFile Nodes for Excel Export**  
    - For Users:  
      - Name: `Convert to Excel Users`  
      - Operation: `xlsx`  
      - FileName: `Zammad_Users.xlsx` (correct the original typo if needed)  
      - Connect output of Users filter or directly from User object node if no filter  
    - For Organizations:  
      - Name: `Convert to Excel Organizations`  
      - Operation: `xlsx`  
      - FileName: `Zammad_Organizations.xlsx`  
      - Connect output of Organization filter or directly  
    - For Roles:  
      - Name: `Convert to Excel Roles`  
      - Operation: `xlsx`  
      - FileName: `Zammad_Roles.xlsx`  
      - Connect output of Role filter or directly  
    - For Groups:  
      - Name: `Convert to Excel Groups`  
      - Operation: `xlsx`  
      - FileName: `Zammad_Groups.xlsx`  
      - Connect output of Group filter or directly

13. **Validate Workflow and Credentials**  
    - Ensure all API credentials are set properly in Credentials section  
    - Replace placeholder base URL and API key with actual values

14. **Run Workflow Manually**  
    - Use manual trigger node to start the workflow  
    - The Excel files will be generated as output of ConvertToFile nodes

---

### 5. General Notes & Resources

| Note Content                                                                                       | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow reads Zammad data via API using both native Zammad nodes and HTTP Request nodes.   | Zammad API documentation: https://docs.zammad.org/en/latest/api/index.html                      |
| Filename for Users Excel file contains a typo: `Zammad_Users.xslx` should be `.xlsx`.            | Correct this in the `Convert to Excel Users` node parameters to avoid file recognition issues.  |
| For API authentication use Token Auth with API key, configured in n8n credentials.                 | Credential setup required before running the workflow                                          |
| Suggestions and issues are tracked on GitHub repository.                                         | https://github.com/Sirhexalot/n8n-Export-Zammad-Objects-Users-Roles-Groups-and-Organizations-to-Excel |
| This workflow is inactive by default; enable it in n8n to execute.                               | Workflow activation required before use                                                        |

---

This structured reference enables complete understanding, reproduction, and modification of the Export Zammad Objects workflow in n8n. It highlights all nodes, their relationships, configurations, and potential edge cases for robust operation.