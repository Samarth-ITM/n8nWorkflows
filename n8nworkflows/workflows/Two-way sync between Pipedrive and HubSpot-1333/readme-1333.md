Two-way sync between Pipedrive and HubSpot

https://n8nworkflows.xyz/workflows/workflow-1333-1747220155024.png


# Two-way sync between Pipedrive and HubSpot

### 1. Workflow Overview

This workflow establishes a two-way synchronization between Pipedrive persons and HubSpot contacts. It is designed for use cases where organizations maintain contact data in both CRMs and want to ensure that new or unique contacts in one system are added to the other automatically.

The workflow runs on a schedule (every minute) and consists of three main logical blocks:

- **1.1 Data Retrieval:** Periodically fetches all persons from Pipedrive and contacts from HubSpot.
- **1.2 Unique Item Identification:** Compares the two datasets to find contacts unique to each system using merge operations that remove matching keys.
- **1.3 Data Synchronization:** Updates each CRM with the unique contacts from the other system.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Retrieval Block

- **Overview:**  
  This block triggers the workflow and pulls the complete lists of persons from Pipedrive and contacts from HubSpot for comparison.

- **Nodes Involved:**  
  - Cron  
  - Pipedrive  
  - HubSpot

- **Node Details:**

  - **Cron**  
    - Type: Cron Trigger  
    - Role: Schedules the workflow to start every minute automatically.  
    - Configuration: Default settings with a 1-minute interval (implicit from description).  
    - Inputs: None (trigger node).  
    - Outputs: Connects to Pipedrive and HubSpot nodes in parallel.  
    - Edge Cases: Cron misconfiguration or disabled workflow could prevent execution.

  - **Pipedrive**  
    - Type: Pipedrive Node  
    - Role: Retrieves all persons from Pipedrive.  
    - Configuration:  
      - Resource: person  
      - Operation: getAll  
      - Return All: true (retrieves complete list)  
      - Additional Fields: none  
      - Credentials: uses stored Pipedrive API credentials.  
    - Inputs: Triggered by Cron.  
    - Outputs: Data passed to Merge nodes.  
    - Edge Cases: API rate limits, authentication errors, network timeout.

  - **HubSpot**  
    - Type: HubSpot Node  
    - Role: Retrieves all contacts from HubSpot.  
    - Configuration:  
      - Resource: contact  
      - Operation: getAll  
      - Return All: true (fetch all contacts)  
      - Additional Fields: none  
      - Credentials: uses stored HubSpot API credentials.  
    - Inputs: Triggered by Cron.  
    - Outputs: Data passed to Merge nodes.  
    - Edge Cases: API authentication failures, pagination issues if data is large, network errors.

---

#### 2.2 Unique Item Identification Block

- **Overview:**  
  This block compares data sets from Pipedrive and HubSpot to identify entries unique to each CRM by removing matching keys based on email addresses.

- **Nodes Involved:**  
  - Merge1  
  - Merge2

- **Node Details:**

  - **Merge1**  
    - Type: Merge Node  
    - Role: Identifies contacts present in HubSpot but not in Pipedrive.  
    - Configuration:  
      - Mode: Remove Key Matches  
      - Property Name 1: identity-profiles[0].identities[0].value (Pipedrive email)  
      - Property Name 2: email[0].value (HubSpot email)  
    - Inputs:  
      - Main Input 1: Data from Pipedrive (index 1)  
      - Main Input 2: Data from HubSpot (index 0)  
    - Outputs: Unique HubSpot contacts passed to Update Pipedrive node.  
    - Edge Cases:  
      - Missing or malformed email fields causing comparison to fail.  
      - Array indices (e.g., [0]) might cause errors if data shape changes.  
      - If multiple emails exist, only the first is considered.

  - **Merge2**  
    - Type: Merge Node  
    - Role: Identifies contacts present in Pipedrive but not in HubSpot.  
    - Configuration:  
      - Mode: Remove Key Matches  
      - Property Name 1: email[0].value (HubSpot email)  
      - Property Name 2: identity-profiles[0].identities[0].value (Pipedrive email)  
    - Inputs:  
      - Main Input 1: Data from HubSpot (index 1)  
      - Main Input 2: Data from Pipedrive (index 0)  
    - Outputs: Unique Pipedrive contacts passed to Update HubSpot node.  
    - Edge Cases: Same as Merge1, with symmetrical risks about email field presence and format.

---

#### 2.3 Data Synchronization Block

- **Overview:**  
  This block updates each CRM with the contacts unique to the other CRM, effectively syncing new contacts detected during the merge.

- **Nodes Involved:**  
  - Update Pipedrive  
  - Update HubSpot

- **Node Details:**

  - **Update Pipedrive**  
    - Type: Pipedrive Node  
    - Role: Adds contacts unique to HubSpot into Pipedrive.  
    - Configuration:  
      - Resource: person  
      - Operation: update (implicit from node name)  
      - Name: Extracted from JSON path $json["properties"]["firstname"]["value"]  
      - Email: Extracted from $json["identity-profiles"][0]["identities"][0]["value"]  
      - Credentials: Pipedrive API credentials.  
    - Inputs: Receives unique contacts from Merge1 output.  
    - Outputs: None (ends branch).  
    - Edge Cases:  
      - Missing firstname or email fields causing update failure.  
      - API limitations or validation errors.  
      - If contact already exists, update operation handling depends on Pipedrive API.

  - **Update HubSpot**  
    - Type: HubSpot Node  
    - Role: Adds contacts unique to Pipedrive into HubSpot.  
    - Configuration:  
      - Resource: contact  
      - Operation: update (implicit)  
      - Email: $json["email"][0]["value"]  
      - FirstName: $json["first_name"]  
      - Credentials: HubSpot API credentials.  
    - Inputs: Receives unique contacts from Merge2 output.  
    - Outputs: None (ends branch).  
    - Edge Cases:  
      - Missing email or first name fields.  
      - API validation errors, rate limits.  
      - Potential duplication if Pipedrive data is incomplete or inconsistent.

---

### 3. Summary Table

| Node Name        | Node Type            | Functional Role                          | Input Node(s)      | Output Node(s)           | Sticky Note                                          |
|------------------|----------------------|----------------------------------------|--------------------|--------------------------|------------------------------------------------------|
| Cron             | Cron Trigger         | Schedule workflow execution every minute | None               | Pipedrive, HubSpot       |                                                      |
| Pipedrive        | Pipedrive            | Retrieve all persons from Pipedrive     | Cron               | Merge1 (index 1), Merge2 (index 0) |                                                      |
| HubSpot          | HubSpot              | Retrieve all contacts from HubSpot      | Cron               | Merge1 (index 0), Merge2 (index 1) |                                                      |
| Merge1           | Merge                | Identify HubSpot-only contacts          | Pipedrive (1), HubSpot (0) | Update Pipedrive          |                                                      |
| Merge2           | Merge                | Identify Pipedrive-only contacts        | HubSpot (1), Pipedrive (0) | Update HubSpot            |                                                      |
| Update Pipedrive | Pipedrive            | Add unique HubSpot contacts to Pipedrive | Merge1             | None                     |                                                      |
| Update HubSpot   | HubSpot              | Add unique Pipedrive contacts to HubSpot | Merge2             | None                     |                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Node**  
   - Type: Cron Trigger  
   - Set schedule to run every 1 minute (default or custom Cron expression)  
   - No credentials needed  
   - Connect its output to two separate nodes (Pipedrive and HubSpot)

2. **Create a Pipedrive Node (Retrieve Persons)**  
   - Type: Pipedrive  
   - Credentials: Select or create Pipedrive API credentials with appropriate scopes  
   - Resource: person  
   - Operation: getAll  
   - Return All: true  
   - Additional Fields: leave empty  
   - Connect input from Cron node  
   - Connect output to two Merge nodes (Merge1 input index 1, Merge2 input index 0)

3. **Create a HubSpot Node (Retrieve Contacts)**  
   - Type: HubSpot  
   - Credentials: Select or create HubSpot API credentials  
   - Resource: contact  
   - Operation: getAll  
   - Return All: true  
   - Additional Fields: leave empty  
   - Connect input from Cron node  
   - Connect output to two Merge nodes (Merge1 input index 0, Merge2 input index 1)

4. **Create Merge1 Node**  
   - Type: Merge  
   - Mode: Remove Key Matches  
   - Property Name 1: identity-profiles[0].identities[0].value (Pipedrive email)  
   - Property Name 2: email[0].value (HubSpot email)  
   - Connect inputs:  
     - Input 1: From Pipedrive node (index 1)  
     - Input 2: From HubSpot node (index 0)  
   - Connect output to Update Pipedrive node

5. **Create Merge2 Node**  
   - Type: Merge  
   - Mode: Remove Key Matches  
   - Property Name 1: email[0].value (HubSpot email)  
   - Property Name 2: identity-profiles[0].identities[0].value (Pipedrive email)  
   - Connect inputs:  
     - Input 1: From HubSpot node (index 1)  
     - Input 2: From Pipedrive node (index 0)  
   - Connect output to Update HubSpot node

6. **Create Update Pipedrive Node**  
   - Type: Pipedrive  
   - Credentials: Use Pipedrive API credentials  
   - Resource: person  
   - Operation: update (or create if needed)  
   - Set Name field using expression: `{{$json["properties"]["firstname"]["value"]}}`  
   - Set Additional Fields: email array with expression: `{{$json["identity-profiles"][0]["identities"][0]["value"]}}`  
   - Connect input from Merge1 node output  
   - No outputs needed (end node)

7. **Create Update HubSpot Node**  
   - Type: HubSpot  
   - Credentials: Use HubSpot API credentials  
   - Resource: contact  
   - Operation: update (or create if needed)  
   - Email field: `{{$json["email"][0]["value"]}}`  
   - Additional Fields: firstName: `{{$json["first_name"]}}`  
   - Connect input from Merge2 node output  
   - No outputs needed (end node)

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                             |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| The workflow screenshot is referenced as "workflow-screenshot" from fileId:568                      | Visual aid for layout, not embedded here                    |
| Email fields are nested and indexed (e.g., email[0].value), ensure data always follows this format  | Important for avoiding expression or runtime errors         |
| Removing key matches based on emails assumes emails are unique identifiers in both CRMs             | Synchronization depends heavily on consistent email usage   |
| API credentials must have sufficient permissions for read and write operations on persons/contacts  | Credential misconfiguration will cause failures             |
| Cron node frequency can be adjusted depending on API rate limits and synchronization needs          | Too frequent runs might hit limits or cause duplication     |
| For troubleshooting, inspect node outputs and errors for missing or malformed fields                 | Helps identify edge cases in data formatting                 |

---

This completes the comprehensive reference document for the "Two-way sync between Pipedrive and HubSpot" workflow.