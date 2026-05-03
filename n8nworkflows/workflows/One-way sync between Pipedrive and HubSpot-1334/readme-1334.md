One-way sync between Pipedrive and HubSpot

https://n8nworkflows.xyz/workflows/one-way-sync-between-pipedrive-and-hubspot-1334


# One-way sync between Pipedrive and HubSpot

### 1. Workflow Overview

This workflow performs a one-way synchronization of person/contact data from Pipedrive to HubSpot. It is designed to run automatically every minute and ensures that any new persons in Pipedrive (i.e., those not already present in HubSpot contacts) are added to HubSpot as contacts.

The workflow is logically organized into the following blocks:

- **1.1 Input Reception and Scheduling:** Initiates the workflow at fixed intervals using a Cron trigger.
- **1.2 Data Retrieval:** Fetches all persons from Pipedrive and all contacts from HubSpot.
- **1.3 Data Comparison:** Compares the two datasets to identify persons in Pipedrive that do not exist in HubSpot.
- **1.4 Data Synchronization:** Adds the unique Pipedrive persons to HubSpot as new contacts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Scheduling

**Overview:**  
This block triggers the workflow to run automatically every minute to keep data synchronized regularly.

**Nodes Involved:**  
- Cron

**Node Details:**

- **Cron**  
  - *Type & Role*: Trigger node; initiates workflow execution based on schedule.  
  - *Configuration*: Configured to trigger every minute (`everyMinute` mode).  
  - *Expressions/Variables*: None.  
  - *Input/Output*: No input; outputs to both Pipedrive and HubSpot nodes in parallel.  
  - *Version Requirements*: Compatible with n8n version 1.0+.  
  - *Potential Failures*: Cron misconfiguration (unlikely), system time issues.  
  - *Sub-workflow*: None.

---

#### 1.2 Data Retrieval

**Overview:**  
This block fetches all persons from Pipedrive and all contacts from HubSpot for comparison.

**Nodes Involved:**  
- Pipedrive  
- Hubspot

**Node Details:**

- **Pipedrive**  
  - *Type & Role*: API integration node; retrieves all persons from Pipedrive.  
  - *Configuration*:  
    - Resource: `person`  
    - Operation: `getAll`  
    - Return all records without pagination limits.  
  - *Expressions/Variables*: None used beyond retrieving all records.  
  - *Input/Output*: Receives input from Cron node; outputs to Merge node (input 0).  
  - *Credentials*: Uses stored Pipedrive API credentials.  
  - *Version Requirements*: Standard for n8n Pipedrive node.  
  - *Potential Failures*: API auth errors, rate limiting, network issues.

- **Hubspot**  
  - *Type & Role*: API integration node; retrieves all contacts from HubSpot.  
  - *Configuration*:  
    - Resource: `contact`  
    - Operation: `getAll`  
    - Return all records without pagination limits.  
  - *Expressions/Variables*: None beyond retrieving all records.  
  - *Input/Output*: Receives input from Cron node; outputs to Merge node (input 1).  
  - *Credentials*: Uses stored HubSpot API credentials.  
  - *Version Requirements*: Standard for n8n HubSpot node.  
  - *Potential Failures*: API auth errors, rate limiting, network issues.

---

#### 1.3 Data Comparison

**Overview:**  
This block compares emails from Pipedrive persons and HubSpot contacts to find persons that exist only in Pipedrive.

**Nodes Involved:**  
- Merge

**Node Details:**

- **Merge**  
  - *Type & Role*: Data operation node; compares two data streams to find unique items in Pipedrive.  
  - *Configuration*:  
    - Mode: `removeKeyMatches` (removes items from first input that have matching keys in second input)  
    - Property Name 1: `email[0].value` (email of Pipedrive person)  
    - Property Name 2: `identity-profiles[0].identities[0].value` (email of HubSpot contact)  
  - *Expressions/Variables*: Uses JSON path expressions to extract email fields for comparison.  
  - *Input/Output*:  
    - Input 0: From Pipedrive node (persons)  
    - Input 1: From Hubspot node (contacts)  
    - Output: Unique Pipedrive persons (not in HubSpot) forwarded to HubSpot2 node.  
  - *Version Requirements*: Standard Merge node functionality.  
  - *Potential Failures*:  
    - Missing or inconsistent email fields in either dataset may cause incorrect matching.  
    - Empty arrays or malformed data could lead to no output or errors.

---

#### 1.4 Data Synchronization

**Overview:**  
This block takes the unique persons identified and creates corresponding contacts in HubSpot.

**Nodes Involved:**  
- HubSpot2

**Node Details:**

- **HubSpot2**  
  - *Type & Role*: API integration node; creates new contacts in HubSpot.  
  - *Configuration*:  
    - Resource: `contact`  
    - Operation: Create (implied by absence of explicit operation, but functionally adding contacts)  
    - Fields mapped:  
      - Email: `{{$json["email"][0]["value"]}}` (email from Pipedrive person)  
      - First Name: `{{$json["first_name"]}}` (first name from Pipedrive person)  
  - *Expressions/Variables*: Uses expressions to map email and first name dynamically from the input JSON.  
  - *Input/Output*: Receives input from Merge node output; no further output connections.  
  - *Credentials*: Uses same HubSpot API credentials as the Hubspot node.  
  - *Version Requirements*: Standard HubSpot node compatible with create operation.  
  - *Potential Failures*:  
    - Missing email or first name causing API errors or incomplete contact creation.  
    - API rate limiting or authentication failure.  
    - Mapping errors if expected JSON structure differs.

---

### 3. Summary Table

| Node Name | Node Type              | Functional Role            | Input Node(s) | Output Node(s) | Sticky Note                                                                                     |
|-----------|------------------------|----------------------------|---------------|----------------|------------------------------------------------------------------------------------------------|
| Cron      | Cron Trigger           | Workflow scheduler          |               | Pipedrive, Hubspot |                                                                                              |
| Pipedrive | Pipedrive API Node     | Retrieve persons from Pipedrive | Cron          | Merge          |                                                                                              |
| Hubspot   | HubSpot API Node       | Retrieve contacts from HubSpot | Cron          | Merge          |                                                                                              |
| Merge     | Merge Node             | Compare and identify unique Pipedrive persons | Pipedrive, Hubspot | HubSpot2       |                                                                                                |
| HubSpot2  | HubSpot API Node       | Add unique Pipedrive persons as contacts in HubSpot | Merge         |                |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Node**  
   - Add a Cron trigger node named `Cron`.  
   - Set it to trigger **every minute** (`Every Minute` mode).  
   - No credentials needed.  

2. **Create Pipedrive Node**  
   - Add a Pipedrive node named `Pipedrive`.  
   - Set Resource to `person`.  
   - Set Operation to `getAll`.  
   - Enable `Return All` to fetch all persons without limit.  
   - Connect input from `Cron` node.  
   - Configure Pipedrive credentials with API key or OAuth as per your account.  

3. **Create HubSpot Node**  
   - Add a HubSpot node named `Hubspot`.  
   - Set Resource to `contact`.  
   - Set Operation to `getAll`.  
   - Enable `Return All` to fetch all contacts.  
   - Connect input from `Cron` node (parallel to Pipedrive).  
   - Configure HubSpot credentials (OAuth2 recommended).  

4. **Create Merge Node**  
   - Add a Merge node named `Merge`.  
   - Set Mode to `Remove Key Matches`.  
   - Set Property Name 1 to `email[0].value` (this accesses the first email value in Pipedrive person JSON).  
   - Set Property Name 2 to `identity-profiles[0].identities[0].value` (this accesses the first email identity in HubSpot contact JSON).  
   - Connect input 0 from `Pipedrive` node.  
   - Connect input 1 from `Hubspot` node.  

5. **Create HubSpot Node for Adding Contacts**  
   - Add a HubSpot node named `HubSpot2`.  
   - Set Resource to `contact`.  
   - Operation should be set to `create` (default when specifying fields).  
   - In Additional Fields, set:  
     - `email` parameter: `={{$json["email"][0]["value"]}}`  
     - `firstName` parameter: `={{$json["first_name"]}}`  
   - Connect input from `Merge` node.  
   - Use same HubSpot credentials as the earlier HubSpot node.  

6. **Connect all nodes as described:**  
   - `Cron` outputs to both `Pipedrive` and `Hubspot` nodes in parallel.  
   - `Pipedrive` connects to `Merge` input 0.  
   - `Hubspot` connects to `Merge` input 1.  
   - `Merge` outputs to `HubSpot2` node.  

7. **Save and activate the workflow** to enable automatic one-way sync every minute.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                 |
|-----------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Workflow screenshot provided (not embedded here) to visualize node arrangement.                      | Original workflow visual reference                             |
| Use the `removeKeyMatches` mode in Merge node to detect unique records by comparing email fields.   | n8n documentation on Merge node modes: https://docs.n8n.io/nodes/n8n-nodes-base.merge/ |
| Ensure API credentials for Pipedrive and HubSpot have appropriate scopes for reading and writing data.| Pipedrive and HubSpot API documentation for auth setup.       |
| Consider adding error handling or logging nodes to monitor API failures or rate limits.             | Best practices for production workflows.                        |

---

This documentation fully describes the workflow architecture, node configuration, and step-by-step reproduction instructions to facilitate maintenance, adaptation, or integration into larger automation systems.