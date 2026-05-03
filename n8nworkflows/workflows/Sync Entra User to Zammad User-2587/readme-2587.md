Sync Entra User to Zammad User

https://n8nworkflows.xyz/workflows/sync-entra-user-to-zammad-user-2587


# Sync Entra User to Zammad User

### 1. Workflow Overview

This workflow automates user synchronization between Microsoft Entra (Azure AD) and Zammad, an open-source helpdesk system. It is designed to keep user data consistent by fetching members of a specified Entra group, transforming the data into a format compatible with Zammad, and then creating, updating, or deactivating users in Zammad based on changes in Entra.

The workflow consists of the following logical blocks:

- **1.1 Input Reception**: Manual trigger to start the synchronization process.
- **1.2 Entra Group Retrieval and Filtering**: Fetch all groups from Entra and filter to the designated synchronization group.
- **1.3 Entra Group Members Retrieval**: Fetch members of the selected Entra group.
- **1.4 Entra User Data Processing**: Flatten and transform the Entra user data into a universal user object tailored for Zammad.
- **1.5 Zammad User Retrieval and Filtering**: Fetch all active users from Zammad who have the custom field `entra_object_type` set to "user".
- **1.6 User Matching and Comparison**: Merge Entra users with Zammad users by email to identify existing, new, and removed users.
- **1.7 User Synchronization Actions**:
  - Update existing Zammad users based on Entra data.
  - Create new Zammad users for Entra users not found in Zammad.
  - Deactivate Zammad users who no longer exist in the Entra group.

---

### 2. Block-by-Block Analysis

#### Block 1.1: Input Reception

- **Overview**: Initiates the workflow manually.
- **Nodes Involved**:  
  - When clicking ‘Test workflow’

##### Node: When clicking ‘Test workflow’  
- Type: Manual Trigger  
- Role: Entry point to start the workflow  
- Configuration: Default manual trigger with no parameters  
- Input: None  
- Output: Triggers both Entra and Zammad data retrieval nodes  
- Version: 1  
- Edge Cases: No trigger means no execution; manual start required

---

#### Block 1.2: Entra Group Retrieval and Filtering

- **Overview**: Retrieves all groups from Entra and selects the specific group named "ENTRA" to synchronize users from.
- **Nodes Involved**:  
  - Get Groups from Entra  
  - Remove outer Array  
  - Select Entra Zammad default Group  
  - Note1 (sticky note)

##### Node: Get Groups from Entra  
- Type: HTTP Request  
- Role: Calls Microsoft Graph API to fetch all groups  
- Configuration:  
  - URL: `https://graph.microsoft.com/v1.0/groups`  
  - Authentication: Microsoft OAuth2 credentials for Entra  
- Input: Trigger from manual start  
- Output: JSON array containing groups  
- Version: 4.2  
- Edge Cases: API rate limits, authentication failures, empty group list

##### Node: Remove outer Array  
- Type: Split Out  
- Role: Extracts the "value" array (list of groups) from API response  
- Configuration: Field to split out: `value`  
- Input: Output of Get Groups from Entra  
- Output: Individual group objects  
- Version: 1  
- Edge Cases: Missing or malformed response

##### Node: Select Entra Zammad default Group  
- Type: If  
- Role: Filters groups to select only the one named exactly "ENTRA"  
- Configuration: Condition: `$json.displayName` equals "ENTRA" (case sensitive)  
- Input: Individual group objects  
- Output: Only the group with displayName "ENTRA"  
- Version: 2.2  
- Edge Cases: Group name not found, multiple groups with same name

##### Sticky Note: Note1  
- Content: "Select Entra Users in a named Entra Group that should be synced to Zammad"  
- Applies to: This block  
- Purpose: Instructional for configuring the group name to synchronize

---

#### Block 1.3: Entra Group Members Retrieval

- **Overview**: Fetches all members of the selected "ENTRA" group from Entra.
- **Nodes Involved**:  
  - Get Members of the default group  
  - Remove outer Array from Entra User  
  - If (check for members existence)

##### Node: Get Members of the default group  
- Type: HTTP Request  
- Role: Calls Microsoft Graph API to get all members of the specified group by ID  
- Configuration:  
  - URL: `https://graph.microsoft.com/v1.0/groups/{{ $json.id }}/members`  
  - Authentication: Microsoft OAuth2 credentials  
- Input: Output from "Select Entra Zammad default Group" node (group object)  
- Output: JSON array of group members  
- Version: 4.2  
- Edge Cases: Group ID invalid, no members, API errors

##### Node: Remove outer Array from Entra User  
- Type: Split Out  
- Role: Unwraps "value" array of members  
- Configuration: Field to split out: `value`  
- Input: Members array response  
- Output: Individual member objects  
- Version: 1  
- Edge Cases: Malformed API response

##### Node: If (check for members existence)  
- Type: If  
- Role: Checks if member data exists (non-empty) to continue processing  
- Configuration: Condition: Checks if `$json` exists (object presence)  
- Input: Individual member objects  
- Output: Passes only if members exist  
- Version: 2.2  
- Edge Cases: No members found, empty dataset

---

#### Block 1.4: Entra User Data Processing

- **Overview**: Transforms raw Entra user data into a normalized user object compatible with Zammad.
- **Nodes Involved**:  
  - Zammad Univeral User Object

##### Node: Zammad Univeral User Object  
- Type: Set  
- Role: Maps key Entra user properties to a universal user object format for Zammad  
- Configuration:  
  - Extract and set fields:  
    - `entra_key` = Entra user `id` (number)  
    - `email` = `userPrincipalName` (string)  
    - `lastname` = `surname` (string)  
    - `firstname` = `givenName` (string)  
    - `mobile` = `mobilePhone` (string)  
    - `phone` = first element of `businessPhones` array (string)  
  - Keeps only these fields  
- Input: Valid Entra member objects  
- Output: Standardized user objects for downstream processing  
- Version: 1  
- Edge Cases: Missing or empty fields, array fields missing elements

---

#### Block 1.5: Zammad User Retrieval and Filtering

- **Overview**: Retrieves all users from Zammad and filters to active users with `entra_object_type` set to "user".
- **Nodes Involved**:  
  - Get Zammad Users  
  - Select only active Users and entra_object_type="user"

##### Node: Get Zammad Users  
- Type: Zammad  
- Role: Fetches all users from Zammad API  
- Configuration:  
  - Operation: getAll  
  - Return all users: true  
  - No filters applied at this step  
  - Credentials: Zammad Token Auth  
- Input: Manual trigger  
- Output: List of all users in Zammad  
- Version: 1  
- Edge Cases: API limits, auth failures

##### Node: Select only active Users and entra_object_type="user"  
- Type: If  
- Role: Filters Zammad users to only those:  
  - With `active` flag true  
  - With custom field `entra_object_type` equal to "user"  
- Configuration: Conditions checking `$json.entra_object_type === "user"` and `$json.active === true`  
- Input: All Zammad users  
- Output: Filtered active Entra-synced users  
- Version: 2.2  
- Edge Cases: Missing custom fields, inactive users

---

#### Block 1.6: User Matching and Comparison

- **Overview**: Merges Entra and Zammad user lists by email, then identifies new and removed users.
- **Nodes Involved**:  
  - Merge  
  - Find new Zammad Users  
  - Find removed Users

##### Node: Merge  
- Type: Merge  
- Role: Combines Entra user objects and filtered Zammad users on the email field  
- Configuration:  
  - Mode: Combine  
  - Match on: `email` (string field)  
- Input:  
  - Left: Entra universal user objects  
  - Right: Filtered Zammad users  
- Output: Merged records indicating matches and unmatched users  
- Version: 3  
- Edge Cases: Duplicate emails, missing emails

##### Node: Find new Zammad Users  
- Type: Compare Datasets  
- Role: Identifies users present in Entra but not in Zammad (new users to create)  
- Configuration:  
  - Merge by fields: Entra `email` and Zammad `email`  
- Input: Merged user records  
- Output: Users to create in Zammad  
- Version: 2.3  
- Edge Cases: Email mismatches

##### Node: Find removed Users  
- Type: Compare Datasets  
- Role: Identifies users present in Zammad but missing in Entra (to be deactivated)  
- Configuration:  
  - Merge by fields: Entra `entra_key` and Zammad `entra_key`  
  - Preference for input from Zammad dataset  
- Input: Merged user records  
- Output: Users to deactivate in Zammad  
- Version: 2.3  
- Edge Cases: Missing keys, inconsistent data

---

#### Block 1.7: User Synchronization Actions

- **Overview**: Performs create, update, or deactivate operations on Zammad users based on comparison results.
- **Nodes Involved**:  
  - Update Zammad User  
  - Create Zammad User  
  - Deactivate Zammad User

##### Node: Update Zammad User  
- Type: Zammad  
- Role: Updates existing users in Zammad with the latest data from Entra  
- Configuration:  
  - Operation: update  
  - User ID: from `$json.id` (Zammad user ID)  
  - Fields updated: phone, mobile, lastname, firstname, custom fields (`entra_key`, `entra_object_type` = "user")  
- Input: Matched users from merge output  
- Output: Updated user records  
- Version: 1  
- Edge Cases: API failures, missing IDs

##### Node: Create Zammad User  
- Type: Zammad  
- Role: Creates new users in Zammad for Entra users not present in Zammad  
- Configuration:  
  - Fields set: lastname, firstname, email, phone, mobile, custom fields (`entra_key`, `entra_object_type` = "user")  
- Input: New users from Find new Zammad Users node  
- Output: Created user records  
- Version: 1  
- Edge Cases: Duplicate user creation, API errors

##### Node: Deactivate Zammad User  
- Type: Zammad  
- Role: Deactivates users in Zammad who are no longer in the Entra group  
- Configuration:  
  - Operation: update  
  - User ID: `$json.id`  
  - Fields updated: active = false, phone, mobile, lastname, firstname, custom field `entra_key`  
- Input: Removed users from Find removed Users node  
- Output: Deactivated user records  
- Version: 1  
- Edge Cases: Failures in update, incorrect deactivation

---

### 3. Summary Table

| Node Name                         | Node Type           | Functional Role                               | Input Node(s)                          | Output Node(s)                           | Sticky Note                                                         |
|----------------------------------|---------------------|----------------------------------------------|--------------------------------------|-----------------------------------------|----------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger      | Manual start trigger                          | None                                 | Get Zammad Users, Get Groups from Entra |                                                                      |
| Get Groups from Entra             | HTTP Request        | Fetch all Entra groups                        | When clicking ‘Test workflow’        | Remove outer Array                       |                                                                      |
| Remove outer Array                | Split Out           | Extract groups array                          | Get Groups from Entra                 | Select Entra Zammad default Group       |                                                                      |
| Select Entra Zammad default Group| If                  | Filter to the Entra group named "ENTRA"      | Remove outer Array                    | Get Members of the default group        | Select Entra Users in a named Entra Group that should be synced to Zammad |
| Get Members of the default group | HTTP Request        | Fetch members of the selected Entra group    | Select Entra Zammad default Group    | Remove outer Array from Entra User      |                                                                      |
| Remove outer Array from Entra User| Split Out           | Extract array of group members                | Get Members of the default group     | If (check for members existence)         |                                                                      |
| If (check for members existence) | If                  | Proceed only if members exist                  | Remove outer Array from Entra User   | Zammad Univeral User Object              |                                                                      |
| Zammad Univeral User Object       | Set                 | Normalize user data for Zammad compatibility | If (check for members existence)     | Merge, Find new Zammad Users, Find removed Users |                                                                      |
| Get Zammad Users                 | Zammad              | Fetch all Zammad users                         | When clicking ‘Test workflow’        | Select only active Users and entra_object_type="user" |                                                                      |
| Select only active Users and entra_object_type="user" | If | Filter active Zammad users synced from Entra | Get Zammad Users                    | Merge, Find new Zammad Users, Find removed Users |                                                                      |
| Merge                           | Merge               | Combine Entra and Zammad users by email       | Zammad Univeral User Object, Select only active Users and entra_object_type="user" | Update Zammad User                       |                                                                      |
| Find new Zammad Users           | Compare Datasets     | Identify Entra users missing in Zammad        | Merge                               | Create Zammad User                       |                                                                      |
| Find removed Users              | Compare Datasets     | Identify Zammad users missing in Entra        | Merge                               | Deactivate Zammad User                   |                                                                      |
| Update Zammad User              | Zammad              | Update existing Zammad users                   | Merge                              |                                         |                                                                      |
| Create Zammad User              | Zammad              | Create new Zammad users                         | Find new Zammad Users                |                                         |                                                                      |
| Deactivate Zammad User          | Zammad              | Deactivate Zammad users no longer in Entra    | Find removed Users                  |                                         |                                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: "When clicking ‘Test workflow’"  
   - No parameters needed.

2. **Add HTTP Request Node to Fetch Entra Groups**  
   - Name: "Get Groups from Entra"  
   - HTTP Method: GET  
   - URL: `https://graph.microsoft.com/v1.0/groups`  
   - Authentication: Microsoft OAuth2 (configure credentials with proper API permissions)  
   - Connect output from manual trigger.

3. **Add Split Out Node to Extract Groups Array**  
   - Name: "Remove outer Array"  
   - Field to split out: `value`  
   - Connect input from "Get Groups from Entra".

4. **Add If Node to Filter Group by Name**  
   - Name: "Select Entra Zammad default Group"  
   - Condition: `$json.displayName` equals "ENTRA" (case sensitive)  
   - Connect input from "Remove outer Array".

5. **Add HTTP Request Node to Fetch Group Members**  
   - Name: "Get Members of the default group"  
   - HTTP Method: GET  
   - URL: `https://graph.microsoft.com/v1.0/groups/{{ $json.id }}/members`  
   - Authentication: Microsoft OAuth2 (same credentials as before)  
   - Connect input from "Select Entra Zammad default Group".

6. **Add Split Out Node to Extract Members Array**  
   - Name: "Remove outer Array from Entra User"  
   - Field to split out: `value`  
   - Connect input from "Get Members of the default group".

7. **Add If Node to Check Members Existence**  
   - Name: "If"  
   - Condition: Object existence check on `$json` (to ensure member data exists)  
   - Connect input from "Remove outer Array from Entra User".

8. **Add Set Node to Normalize Entra User Data**  
   - Name: "Zammad Univeral User Object"  
   - Set fields (keep only these):  
     - `entra_key` = `$json.id` (number)  
     - `email` = `$json.userPrincipalName` (string)  
     - `lastname` = `$json.surname` (string)  
     - `firstname` = `$json.givenName` (string)  
     - `mobile` = `$json.mobilePhone` (string)  
     - `phone` = `$json.businessPhones[0]` (string)  
   - Connect input from "If".

9. **Add Zammad Node to Fetch All Users**  
   - Name: "Get Zammad Users"  
   - Operation: getAll users  
   - Return all: true  
   - Credentials: Zammad Token Auth (configure with API access)  
   - Connect input from manual trigger.

10. **Add If Node to Filter Active Entra-Synced Zammad Users**  
    - Name: "Select only active Users and entra_object_type=\"user\""  
    - Conditions:  
      - `$json.entra_object_type` equals "user"  
      - `$json.active` is true  
    - Connect input from "Get Zammad Users".

11. **Add Merge Node to Combine Entra and Zammad Users**  
    - Name: "Merge"  
    - Mode: Combine  
    - Match on field: `email`  
    - Left input: "Zammad Univeral User Object" output  
    - Right input: "Select only active Users and entra_object_type=\"user\"" output

12. **Add Compare Datasets Node to Find New Zammad Users**  
    - Name: "Find new Zammad Users"  
    - Merge by fields: Entra `email` and Zammad `email`  
    - Connect input from "Merge".

13. **Add Compare Datasets Node to Find Removed Users**  
    - Name: "Find removed Users"  
    - Merge by fields: Entra `entra_key` and Zammad `entra_key`  
    - Resolve: prefer input 1 (Zammad dataset)  
    - Connect input from "Merge".

14. **Add Zammad Node to Update Existing Users**  
    - Name: "Update Zammad User"  
    - Operation: update user  
    - User ID: `$json.id` (Zammad user id)  
    - Update fields: phone, mobile, lastname, firstname, custom fields `entra_key` and `entra_object_type` ("user")  
    - Credentials: Zammad Token Auth  
    - Connect input from "Merge" node output matching existing users.

15. **Add Zammad Node to Create New Users**  
    - Name: "Create Zammad User"  
    - Create user with fields: lastname, firstname, email, phone, mobile, custom fields `entra_key` and `entra_object_type` ("user")  
    - Credentials: Zammad Token Auth  
    - Connect input from "Find new Zammad Users" node output.

16. **Add Zammad Node to Deactivate Removed Users**  
    - Name: "Deactivate Zammad User"  
    - Operation: update user  
    - User ID: `$json.id`  
    - Set `active` to false and update phone, mobile, lastname, firstname, custom field `entra_key`  
    - Credentials: Zammad Token Auth  
    - Connect input from "Find removed Users" node output.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow is customizable for additional fields or different Entra group names.          | Workflow description                                                                              |
| Requires Microsoft OAuth2 credentials with permissions to read groups and members in Entra.  | Setup instructions                                                                                |
| Requires Zammad API credentials with rights to manage users, including custom fields.        | Setup instructions                                                                                |
| Custom field `entra_key` of type String must exist on Zammad User object.                    | Prerequisites                                                                                     |
| Report issues or suggest improvements at: [Github](https://github.com/Sirhexalot/n8n-Zammad-Sync-Entra-User-to-Zammad-User) | Workflow metadata and support                                                                     |
| Sticky note reminder: "Select Entra Users in a named Entra Group that should be synced to Zammad" | Helps identify configurable synchronization group name                                           |

---

This documentation provides detailed insights and stepwise instructions for understanding, reproducing, and maintaining the "Sync Entra User to Zammad User" workflow in n8n. It anticipates potential API or data issues and guides credential setup for seamless integration.