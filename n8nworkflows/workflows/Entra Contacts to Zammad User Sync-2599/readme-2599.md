Entra Contacts to Zammad User Sync

https://n8nworkflows.xyz/workflows/entra-contacts-to-zammad-user-sync-2599


# Entra Contacts to Zammad User Sync

---

### 1. Workflow Overview

**Purpose:**  
This workflow enables automated synchronization of user contact data between Microsoft Entra (Azure AD contacts) and Zammad, a customer support system. It ensures that Zammad user records reflect the current state of Entra contacts by creating new users, updating existing ones, and deactivating users no longer present in Entra.

**Target Use Cases:**
- IT administrators who want to keep Zammad support users aligned with organizational contacts.
- Customer management teams needing accurate and updated user profiles in Zammad.
- Scenarios requiring regular data consistency between Microsoft Entra and Zammad without manual intervention.

**Logical Blocks:**

- **1.1 Execution Trigger:** Manual start of the sync process.
- **1.2 Data Retrieval:** Fetch all contacts from Entra and all users from Zammad.
- **1.3 Data Preparation:** Filter and transform Entra contacts into a universal user object compatible with Zammad.
- **1.4 Data Matching & Comparison:** Compare Entra contacts with Zammad users to identify new, existing (to update), and removed users.
- **1.5 User Synchronization:** Update existing Zammad users, create new users, or deactivate users no longer in Entra.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Execution Trigger

- **Overview:**  
Starts the workflow manually, allowing the user to test or run the synchronization on demand.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow.  
    - Configuration: No parameters; simple manual activation.  
    - Inputs: None  
    - Outputs: Triggers downstream nodes "Get Zammad Users" and "Get Contacts from Entra".  
    - Edge Cases: None significant; manual trigger avoids automatic execution pitfalls.

---

#### 2.2 Data Retrieval

- **Overview:**  
Fetches current user data from both systems: all existing Zammad users and all contacts from Microsoft Entra.

- **Nodes Involved:**  
  - Get Zammad Users  
  - Get Contacts from Entra  
  - Entra Contacts (splitOut node)

- **Node Details:**

  - **Get Zammad Users**  
    - Type: Zammad node  
    - Role: Retrieves all user records from Zammad via API.  
    - Configuration: Operation "getAll", returns all users without filters.  
    - Credentials: Uses Zammad Token Auth.  
    - Inputs: Manual trigger node  
    - Outputs: Passes user data to "Filter if needed" node.  
    - Edge Cases: API auth errors, rate limits, incomplete data if Zammad API changes.

  - **Get Contacts from Entra**  
    - Type: HTTP Request  
    - Role: Calls Microsoft Graph API to get all contacts from Entra.  
    - Configuration: URL `https://graph.microsoft.com/v1.0/contacts`, authentication via Microsoft OAuth2 credentials.  
    - Credentials: Microsoft OAuth2 API.  
    - Inputs: Manual trigger node  
    - Outputs: Sends data to "Entra Contacts" node.  
    - Edge Cases: API token expiration, permission errors, network timeouts.

  - **Entra Contacts**  
    - Type: SplitOut  
    - Role: Splits the array of contacts from Entra into individual items for processing.  
    - Configuration: Splits on field "value", which contains the array of contacts.  
    - Inputs: Output from "Get Contacts from Entra"  
    - Outputs: Passes each contact individually to "Filter contacts if needed".  
    - Edge Cases: Empty contacts array, malformed response.

---

#### 2.3 Data Preparation

- **Overview:**  
Filters and transforms Entra contacts data into a uniform format suitable for comparison and integration into Zammad.

- **Nodes Involved:**  
  - Filter contacts if needed  
  - Zammad Universal User Object

- **Node Details:**

  - **Filter contacts if needed**  
    - Type: If  
    - Role: Filters out any contacts without data or that do not meet criteria.  
    - Configuration: Checks if the JSON object exists (non-null).  
    - Inputs: Single contact from "Entra Contacts"  
    - Outputs: Passes valid contacts to "Zammad Universal User Object".  
    - Edge Cases: Contacts missing key fields, null or empty items.

  - **Zammad Universal User Object**  
    - Type: Set  
    - Role: Maps Entra contact fields to a standardized user object for Zammad.  
    - Configuration:  
      - Constructs fields:  
        - `entra_key` = Entra contact ID  
        - `email` = contact mail  
        - `lastname` = surname  
        - `firstname` = givenName  
        - `mobile` = second phone number in array (index 1)  
        - `phone` = third phone number in array (index 2)  
      - Keeps only these mapped fields.  
    - Inputs: Filtered Entra contacts  
    - Outputs: Passes universal user object downstream for merging and comparison.  
    - Edge Cases: Missing phone entries (index out of bounds), missing email, inconsistent data types.

---

#### 2.4 Data Matching & Comparison

- **Overview:**  
Compares transformed Entra contacts against existing Zammad users to identify which users need updating, which are new, and which have been removed.

- **Nodes Involved:**  
  - Filter if needed  
  - Merge  
  - Find new Zammad Users  
  - Find removed Users

- **Node Details:**

  - **Filter if needed**  
    - Type: If  
    - Role: Filters Zammad users to those relevant for sync, i.e., users with `entra_object_type` = "contact" and active status true.  
    - Configuration:  
      - Condition 1: `entra_object_type` equals "contact"  
      - Condition 2: `active` is true  
      - Both conditions must be met (AND).  
    - Inputs: Output from "Get Zammad Users"  
    - Outputs: Passes filtered users to "Merge" and comparison nodes.  
    - Edge Cases: Missing custom fields, inactive users filtered out.

  - **Merge**  
    - Type: Merge  
    - Role: Combines the two datasets (Entra universal users and filtered Zammad users) based on matching "email" fields.  
    - Configuration: Mode "combine" with field matching on "email".  
    - Inputs:  
      - Input 1: Zammad filtered users  
      - Input 2: Entra universal user objects  
    - Outputs: Passes merged data to "Update Zammad User".  
    - Edge Cases: Duplicate emails, missing emails causing unmatched records.

  - **Find new Zammad Users**  
    - Type: Compare Datasets  
    - Role: Identifies users present in Entra but not in Zammad (by email), indicating new users to be created.  
    - Configuration: Compare on "email" field in both datasets.  
    - Inputs:  
      - Input 1: Entra universal users  
      - Input 2: Filtered Zammad users  
    - Outputs: Passes new users to "Create Zammad User".  
    - Edge Cases: Email mismatches, email case sensitivity.

  - **Find removed Users**  
    - Type: Compare Datasets  
    - Role: Identifies users present in Zammad but no longer in Entra (by `entra_key`), indicating users to deactivate.  
    - Configuration: Compare on `entra_key` field, prefer input 1 dataset.  
    - Inputs:  
      - Input 1: Filtered Zammad users  
      - Input 2: Entra universal users  
    - Outputs: Passes removed users to "Deactivate Zammad User".  
    - Edge Cases: Missing `entra_key` in Zammad users, mismatched keys.

---

#### 2.5 User Synchronization

- **Overview:**  
Executes updates, creations, or deactivations in Zammad based on comparison results.

- **Nodes Involved:**  
  - Update Zammad User  
  - Create Zammad User  
  - Deactivate Zammad User

- **Node Details:**

  - **Update Zammad User**  
    - Type: Zammad  
    - Role: Updates existing Zammad user records to reflect Entra data.  
    - Configuration:  
      - Operation: update  
      - Uses Zammad user `id` for update  
      - Updates fields: phone, mobile, lastname, firstname  
      - Sets custom fields `entra_key` and `entra_object_type` ("contact")  
    - Inputs: Output from "Merge" node (matched users)  
    - Outputs: None (end of update path)  
    - Edge Cases: API update failures, partial data updates, invalid custom field values.

  - **Create Zammad User**  
    - Type: Zammad  
    - Role: Creates new users in Zammad for Entra contacts not yet present.  
    - Configuration:  
      - Uses lastname, firstname as main fields  
      - Additional fields: email, phone, mobile  
      - Custom fields: `entra_key`, `entra_object_type` ("contact")  
    - Inputs: Output from "Find new Zammad Users" node  
    - Outputs: None (end of creation path)  
    - Edge Cases: Duplicate entries if email conflicts occur, missing required fields, API quota limits.

  - **Deactivate Zammad User**  
    - Type: Zammad  
    - Role: Marks Zammad users as inactive when they no longer exist in Entra.  
    - Configuration:  
      - Updates user by `id`  
      - Sets `active` field to false  
      - Updates other fields (phone, mobile, lastname, firstname, custom fields) for completeness.  
    - Inputs: Output from "Find removed Users"  
    - Outputs: None (end of deactivation path)  
    - Edge Cases: API update failures, users already inactive, missing user IDs.

---

### 3. Summary Table

| Node Name                 | Node Type              | Functional Role                                     | Input Node(s)                     | Output Node(s)                        | Sticky Note                                                                                 |
|---------------------------|------------------------|----------------------------------------------------|----------------------------------|-------------------------------------|--------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger         | Starts the workflow manually                        | None                             | Get Zammad Users, Get Contacts from Entra |                                                                                            |
| Get Zammad Users          | Zammad                 | Retrieves all users from Zammad                     | When clicking ‘Test workflow’    | Filter if needed                    |                                                                                            |
| Get Contacts from Entra   | HTTP Request           | Fetches contacts from Microsoft Entra               | When clicking ‘Test workflow’    | Entra Contacts                     |                                                                                            |
| Entra Contacts            | SplitOut               | Splits Entra contacts array into individual items   | Get Contacts from Entra           | Filter contacts if needed          |                                                                                            |
| Filter contacts if needed | If                     | Filters invalid or empty contacts                    | Entra Contacts                   | Zammad Universal User Object       |                                                                                            |
| Zammad Universal User Object | Set                    | Maps Entra contact fields to universal Zammad user | Filter contacts if needed        | Merge, Find new Zammad Users, Find removed Users |                                                                                            |
| Filter if needed          | If                     | Filters Zammad users to active contacts             | Get Zammad Users                 | Merge, Find new Zammad Users, Find removed Users |                                                                                            |
| Merge                     | Merge                  | Combines Entra and Zammad users by email            | Filter if needed, Zammad Universal User Object | Update Zammad User                |                                                                                            |
| Find new Zammad Users     | Compare Datasets       | Finds contacts in Entra not in Zammad                | Zammad Universal User Object, Filter if needed | Create Zammad User              |                                                                                            |
| Find removed Users        | Compare Datasets       | Finds users in Zammad missing in Entra               | Filter if needed, Zammad Universal User Object | Deactivate Zammad User          |                                                                                            |
| Update Zammad User        | Zammad                 | Updates existing Zammad users with Entra data       | Merge                           | None                              |                                                                                            |
| Create Zammad User        | Zammad                 | Creates new Zammad users from Entra contacts         | Find new Zammad Users           | None                              |                                                                                            |
| Deactivate Zammad User    | Zammad                 | Deactivates users no longer present in Entra         | Find removed Users              | None                              |                                                                                            |
| Note1                     | Sticky Note            | Instructional note: "Select Entra Contacts that should be synced to Zammad" | None                             | None                              | ## Select Entra Contacts that should be synced to Zammad                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Name: `When clicking ‘Test workflow’`  
   - No parameters. This node starts the workflow on manual action.

2. **Create Zammad Get All Users Node**  
   - Type: Zammad  
   - Name: `Get Zammad Users`  
   - Operation: `getAll`  
   - Return All: `true`  
   - Credentials: Configure with Zammad Token Auth credentials.  
   - Connect input from Manual Trigger.

3. **Create HTTP Request Node for Entra Contacts**  
   - Type: HTTP Request  
   - Name: `Get Contacts from Entra`  
   - Method: GET  
   - URL: `https://graph.microsoft.com/v1.0/contacts`  
   - Authentication: Microsoft OAuth2  
   - Credentials: Set up Microsoft OAuth2 API credentials with appropriate permissions for contacts.  
   - Connect input from Manual Trigger.

4. **Create SplitOut Node**  
   - Type: SplitOut  
   - Name: `Entra Contacts`  
   - Field to Split Out: `value` (the array of contacts received from Entra)  
   - Connect input from `Get Contacts from Entra`.

5. **Create If Node to Filter Contacts**  
   - Type: If  
   - Name: `Filter contacts if needed`  
   - Condition: Check if the item exists (JSON object exists).  
   - Connect input from `Entra Contacts`.

6. **Create Set Node to Map Entra Contact to Zammad User Object**  
   - Type: Set  
   - Name: `Zammad Universal User Object`  
   - Fields to set:  
     - `entra_key` = `{{$json["id"]}}`  
     - `email` = `{{$json["mail"]}}`  
     - `lastname` = `{{$json["surname"]}}`  
     - `firstname` = `{{$json["givenName"]}}`  
     - `mobile` = `{{$json["phones"][1]["number"]}}` (handle missing gracefully)  
     - `phone` = `{{$json["phones"][2]["number"]}}` (handle missing gracefully)  
   - Keep only these fields.  
   - Connect input from `Filter contacts if needed`.

7. **Create If Node to Filter Relevant Zammad Users**  
   - Type: If  
   - Name: `Filter if needed`  
   - Conditions (AND):  
     - `entra_object_type` equals `"contact"`  
     - `active` equals `true`  
   - Connect input from `Get Zammad Users`.

8. **Create Merge Node**  
   - Type: Merge  
   - Name: `Merge`  
   - Mode: Combine  
   - Fields to match: `email`  
   - Connect two inputs:  
     - Input 1: Output from `Filter if needed` (Zammad users)  
     - Input 2: Output from `Zammad Universal User Object` (Entra contacts)

9. **Create Compare Datasets Node to Find New Users**  
   - Type: Compare Datasets  
   - Name: `Find new Zammad Users`  
   - Merge by fields: `email` from both datasets  
   - Connect inputs:  
     - Input 1: Entra universal user objects  
     - Input 2: Filtered Zammad users  
   - Configure to identify users in Entra not in Zammad.

10. **Create Compare Datasets Node to Find Removed Users**  
    - Type: Compare Datasets  
    - Name: `Find removed Users`  
    - Merge by fields: `entra_key` from both datasets  
    - Resolve: Prefer input 1 (Zammad users)  
    - Connect inputs:  
      - Input 1: Filtered Zammad users  
      - Input 2: Entra universal user objects  
    - Configure to identify users in Zammad missing in Entra.

11. **Create Zammad Node to Update Users**  
    - Type: Zammad  
    - Name: `Update Zammad User`  
    - Operation: Update  
    - ID: `{{$json["id"]}}` (Zammad user id)  
    - Update Fields:  
      - `phone`, `mobile`, `lastname`, `firstname` from Entra data  
      - Custom fields:  
        - `entra_key` = `{{$json["entra_key"]}}`  
        - `entra_object_type` = `"contact"`  
    - Credentials: Zammad Token Auth  
    - Connect input from `Merge`.

12. **Create Zammad Node to Create New Users**  
    - Type: Zammad  
    - Name: `Create Zammad User`  
    - Operation: Create (default)  
    - Fields:  
      - `lastname`, `firstname`  
      - Additional: `email`, `phone`, `mobile`  
      - Custom fields:  
        - `entra_key` = `{{$json["entra_key"]}}`  
        - `entra_object_type` = `"contact"`  
    - Credentials: Zammad Token Auth  
    - Connect input from `Find new Zammad Users`.

13. **Create Zammad Node to Deactivate Users**  
    - Type: Zammad  
    - Name: `Deactivate Zammad User`  
    - Operation: Update  
    - ID: `{{$json["id"]}}`  
    - Update Fields:  
      - `active` = `false`  
      - `phone`, `mobile`, `lastname`, `firstname`  
      - Custom fields:  
        - `entra_key` = `{{$json["entra_key"]}}`  
        - `entra_object_type` = `"contact"`  
    - Credentials: Zammad Token Auth  
    - Connect input from `Find removed Users`.

14. **Add Sticky Note**  
    - Content: "## Select Entra Contacts that should be synced to Zammad"  
    - Position as needed for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow requires a custom field `entra_key` of type `String` in Zammad's User Object.             | Zammad customization prerequisite                                                                                 |
| Another custom field `entra_object_type` (Single selection with keys: user, contact) is required in Zammad. | Zammad customization prerequisite                                                                                 |
| GitHub repository for issues or suggestions: https://github.com/Sirhexalot/n8n-Zammad-Sync-Entra-Contacts-to-Zammad-User | Workflow source and support link                                                                                   |
| The workflow is designed for manual execution but can be automated with schedules in n8n.                | Implementation note                                                                                               |
| Microsoft OAuth2 credentials must have permissions to read contacts from Microsoft Graph API.            | Microsoft Entra integration prerequisite                                                                          |
| Zammad Token Auth API credentials must have user management rights.                                     | Zammad API integration prerequisite                                                                               |

---

This document provides a detailed reference for the "Entra Contacts to Zammad User Sync" workflow, enabling full understanding, reproduction, and customization.