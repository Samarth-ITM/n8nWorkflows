Create, update, and get activity in Strava

https://n8nworkflows.xyz/workflows/create--update--and-get-activity-in-strava-744


# Create, update, and get activity in Strava

### 1. Workflow Overview

This workflow automates the process of creating, updating, and retrieving an activity in Strava via its API. It is designed for users who want to programmatically manage Strava activities in a sequential manner—first creating a new running activity, then updating its details, and finally fetching the updated activity data for confirmation or further use.

**Logical Blocks:**

- **1.1 Input Trigger:** Manual trigger to start the workflow.
- **1.2 Create Activity:** Creates a new running activity in Strava with predefined parameters.
- **1.3 Update Activity:** Updates the newly created activity with additional descriptive information.
- **1.4 Get Activity:** Retrieves the updated activity data to confirm changes or use downstream.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger

- **Overview:**  
  Serves as the entry point to manually start the workflow execution.

- **Nodes Involved:**  
  - On clicking 'execute'

- **Node Details:**  
  - **Type & Role:** Manual Trigger node; initiates the workflow on user command.  
  - **Configuration:** No parameters; triggers execution once clicked.  
  - **Key Expressions/Variables:** None.  
  - **Input/Output Connections:**  
    - Output: Connected to the "Strava" node (Create Activity).  
  - **Version Requirements:** Compatible with all n8n versions supporting Manual Trigger (version 1 here).  
  - **Edge Cases / Failures:** No external dependencies; minimal failure risk unless user does not trigger manually.  
  - **Sub-workflow:** None.

#### 1.2 Create Activity

- **Overview:**  
  Creates a new Strava activity with predefined details such as name, type, start date, duration, and distance.

- **Nodes Involved:**  
  - Strava

- **Node Details:**  
  - **Type & Role:** Strava node configured for "create" operation (default). It sends an API request to create a new activity.  
  - **Configuration Choices:**  
    - Name: "Morning Run"  
    - Type: "Run"  
    - Start Date: October 1, 2020, 18:30 UTC  
    - Elapsed Time: 3600 seconds (1 hour)  
    - Distance: 1000 meters (via additionalFields)  
  - **Key Expressions/Variables:** None (static values).  
  - **Input/Output Connections:**  
    - Input: From Manual Trigger  
    - Output: To "Strava1" (Update Activity)  
  - **Version Requirements:** Requires valid Strava OAuth2 credentials configured as "stravaOAuth2Api".  
  - **Edge Cases / Failures:**  
    - OAuth token expired or invalid  
    - API rate limits exceeded  
    - Invalid or missing required fields causing API rejection  
  - **Sub-workflow:** None.

#### 1.3 Update Activity

- **Overview:**  
  Updates the description field of the activity created by the previous node.

- **Nodes Involved:**  
  - Strava1

- **Node Details:**  
  - **Type & Role:** Strava node configured for "update" operation; modifies existing activity data.  
  - **Configuration Choices:**  
    - Operation: "update"  
    - Activity ID: Dynamically set from the output of the "Strava" node (`{{$node["Strava"].json["id"]}}`).  
    - Update Fields: Sets description to "Morning run in the park".  
  - **Key Expressions/Variables:**  
    - Activity ID dynamically obtained from previous node's JSON output.  
  - **Input/Output Connections:**  
    - Input: From "Strava" (Create Activity)  
    - Output: To "Strava2" (Get Activity)  
  - **Version Requirements:** Same OAuth2 credential requirements as Create Activity node.  
  - **Edge Cases / Failures:**  
    - Invalid or expired OAuth token  
    - Activity ID does not exist or is inaccessible  
    - API rate limits  
    - Expression failure if previous node output is missing or malformed  
  - **Sub-workflow:** None.

#### 1.4 Get Activity

- **Overview:**  
  Retrieves the full data of the updated activity to verify or use the latest information.

- **Nodes Involved:**  
  - Strava2

- **Node Details:**  
  - **Type & Role:** Strava node configured for "get" operation; fetches activity details by ID.  
  - **Configuration Choices:**  
    - Operation: "get"  
    - Activity ID: Dynamically set from "Strava" node output (`{{$node["Strava"].json["id"]}}`).  
  - **Key Expressions/Variables:**  
    - Activity ID expression identical to Update Activity node.  
  - **Input/Output Connections:**  
    - Input: From "Strava1" (Update Activity)  
    - Output: None (end of workflow)  
  - **Version Requirements:** Requires valid Strava OAuth2 credentials.  
  - **Edge Cases / Failures:**  
    - Token expiration or invalidation  
    - Activity ID not found  
    - Expression failure if IDs are missing  
  - **Sub-workflow:** None.

---

### 3. Summary Table

| Node Name           | Node Type           | Functional Role          | Input Node(s)           | Output Node(s)           | Sticky Note                      |
|---------------------|---------------------|-------------------------|------------------------|--------------------------|---------------------------------|
| On clicking 'execute'| Manual Trigger      | Entry point to start workflow | None                  | Strava                   |                                 |
| Strava              | Strava              | Create new Strava activity | On clicking 'execute'  | Strava1                  |                                 |
| Strava1             | Strava              | Update created activity  | Strava                 | Strava2                  |                                 |
| Strava2             | Strava              | Retrieve updated activity| Strava1                | None                     |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Add a Manual Trigger node named "On clicking 'execute'".  
   - No parameters to configure.

2. **Create Strava Node for Activity Creation:**  
   - Add a Strava node named "Strava".  
   - Set operation to default (create).  
   - Configure parameters:  
     - Name: "Morning Run"  
     - Type: "Run"  
     - Start Date: "2020-10-01T18:30:00.000Z" (ISO 8601 format)  
     - Elapsed Time: 3600 (seconds)  
     - Additional Fields: Distance = 1000 (meters)  
   - Assign Strava OAuth2 credentials (must be preconfigured in n8n as "stravaOAuth2Api").

3. **Connect Manual Trigger to Strava (Create) Node:**  
   - Link the output of "On clicking 'execute'" to input of "Strava".

4. **Create Strava Node for Activity Update:**  
   - Add a new Strava node named "Strava1".  
   - Set operation to "update".  
   - Set Activity ID using expression: `{{$node["Strava"].json["id"]}}` (retrieve ID from create node).  
   - In update fields, set Description to "Morning run in the park".  
   - Use the same Strava OAuth2 credentials as before.

5. **Connect "Strava" node output to "Strava1" input.**

6. **Create Strava Node for Activity Retrieval:**  
   - Add a Strava node named "Strava2".  
   - Set operation to "get".  
   - Set Activity ID using expression: `{{$node["Strava"].json["id"]}}`.  
   - Use the same Strava OAuth2 credentials.

7. **Connect "Strava1" output to "Strava2" input.**

8. **Save and activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                              |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Strava API OAuth2 credentials must be configured in n8n beforehand with proper scopes enabled.  | n8n Credential Settings for Strava OAuth2                    |
| Workflow assumes the activity creation succeeds and returns an ID; errors in creation break flow.| Consider adding error handling for robustness                |
| For details on Strava API activity fields, see: https://developers.strava.com/docs/reference/#api-Activities-createActivity | Strava API Documentation                                     |

---

This documentation fully describes the workflow "Create, update, and get activity in Strava" to enable reproduction, modification, and error anticipation for advanced users or automation agents.