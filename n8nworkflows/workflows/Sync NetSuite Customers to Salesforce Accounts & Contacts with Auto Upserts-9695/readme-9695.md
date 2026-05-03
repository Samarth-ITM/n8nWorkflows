Sync NetSuite Customers to Salesforce Accounts & Contacts with Auto Upserts

https://n8nworkflows.xyz/workflows/sync-netsuite-customers-to-salesforce-accounts---contacts-with-auto-upserts-9695


# Sync NetSuite Customers to Salesforce Accounts & Contacts with Auto Upserts

### 1. Workflow Overview

This workflow automates the synchronization of customer records from NetSuite to Salesforce, specifically transforming NetSuite Customers into Salesforce Accounts and Contacts. It supports both company and individual customer types, ensuring that companies are upserted as Salesforce Accounts, and individuals as Accounts with related Contacts. The synchronization handles pagination to process multiple batches of records efficiently and supports delta updates based on the last workflow execution date.

Logical blocks included:

- **1.1 Trigger & Initialization:** Starts the workflow on a schedule and initializes pagination.
- **1.2 NetSuite Data Retrieval:** Retrieves batches of Customer records from NetSuite, either all records or only those changed since the last run.
- **1.3 Customer Record Processing:** Iterates over each customer record, fetching detailed data.
- **1.4 Customer Type Decision:** Determines if the customer is a company or an individual.
- **1.5 Salesforce Upserts:** Upserts company customers as Salesforce Accounts, and individual customers as Accounts and Contacts.
- **1.6 Pagination & Loop Control:** Updates pagination parameters and controls looping until all records are processed.
- **1.7 Workflow Completion:** Marks the end of the workflow once all records are processed.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Initialization

**Overview:**  
This block triggers the workflow on a daily schedule and initializes pagination variables to prepare for batch processing of NetSuite records.

**Nodes Involved:**  
- Execute Workflow Daily  
- Init Offset  
- Has More Records?  
- Retrieve Paging Offset and LastExportDate

**Node Details:**

- **Execute Workflow Daily**  
  - Type: Schedule Trigger  
  - Configuration: Daily interval trigger to run the workflow once every day.  
  - Inputs: None (trigger node)  
  - Outputs: Initiates the workflow chain to Init Offset.  
  - Edge cases: Misconfigured schedule may cause missed runs or duplicate executions.

- **Init Offset**  
  - Type: Code  
  - Role: Initializes static data (`offset=0`, `hasMore=true`) to start batch processing from the beginning.  
  - Outputs: Passes offset and hasMore flag to the condition node.  
  - Edge cases: Static data persistence failure could cause incorrect offsets.

- **Has More Records?**  
  - Type: If  
  - Role: Checks if more records remain to be processed (based on `hasMore` flag).  
  - Condition: Boolean check if `hasMore` is true.  
  - Outputs: If true, proceeds to retrieve offset and last export date; if false, ends workflow.  
  - Edge cases: Incorrect flag handling might prematurely terminate or loop infinitely.

- **Retrieve Paging Offset and LastExportDate**  
  - Type: Code  
  - Role: Retrieves current `offset` and `lastExportDate` from workflow static data for use in data retrieval.  
  - Outputs: Passes offset and lastExportDate to NetSuite data retrieval nodes.  
  - Edge cases: If static data is missing or corrupted, may cause errors in data retrieval.

---

#### 1.2 NetSuite Data Retrieval

**Overview:**  
Fetches batches of customer records from NetSuite using REST API, paginated by offset and limited to 20 records per batch. Supports full retrieval or delta retrieval based on last export date.

**Nodes Involved:**  
- NS: Customer - Get All records  
- NS: Customer - Get Delta records  
- Split Customers Array

**Node Details:**

- **NS: Customer - Get All records**  
  - Type: NetSuite REST API node  
  - Role: Retrieves 20 customers starting from specified offset, without date filters.  
  - Parameters: Limit 20, offset from static data.  
  - Credentials: NetSuite REST OAuth2  
  - Outputs: JSON array of customer records with pagination info.  
  - Edge cases: API rate limits, auth token expiration, network errors.

- **NS: Customer - Get Delta records**  
  - Type: NetSuite REST API node  
  - Role: Retrieves 20 customers updated on or after `lastExportDate` with offset pagination.  
  - Parameters: Uses NetSuite query (Q) parameter with lastModifiedDate filter; limit and offset.  
  - Credentials: Same as above.  
  - Outputs: Similar to "Get All" but filtered for delta updates.  
  - Edge cases: Improper date format may cause query failure.

- **Split Customers Array**  
  - Type: Split Out  
  - Role: Splits batch array into individual customer records for processing one by one.  
  - Inputs: JSON array of customers.  
  - Outputs: Individual customer JSON objects.  
  - Edge cases: Empty arrays lead to no output; malformed input breaks splitting.

---

#### 1.3 Customer Record Processing

**Overview:**  
For each individual customer, fetches detailed record data from NetSuite.

**Nodes Involved:**  
- NS: Customer - Get record

**Node Details:**

- **NS: Customer - Get record**  
  - Type: NetSuite REST API node  
  - Role: Fetches full details for a customer by internal ID.  
  - Parameters: Uses customer `id` from split item.  
  - Credentials: NetSuite REST OAuth2  
  - Outputs: Detailed customer JSON.  
  - Edge cases: Missing ID or record not found; API errors.

---

#### 1.4 Customer Type Decision

**Overview:**  
Decides whether the current customer is a company or an individual to route records appropriately in Salesforce.

**Nodes Involved:**  
- Is Company?

**Node Details:**

- **Is Company?**  
  - Type: If  
  - Role: Checks if the customer is a company (`isPerson` field is false) or a person (`isPerson` true).  
  - Condition: Boolean check on `isPerson` property.  
  - Outputs: True branch for company, false branch for person.  
  - Edge cases: Missing field or inconsistent data may misroute records.

---

#### 1.5 Salesforce Upserts

**Overview:**  
Upserts customer data into Salesforce. Companies are upserted as Accounts; individuals are upserted as Accounts and Contacts linked to those Accounts.

**Nodes Involved:**  
- SF: Create or Update Company Account  
- SF: Create or Update Person's Account  
- SF: Create or Update Contact  
- Merge Company/Contact Results

**Node Details:**

- **SF: Create or Update Company Account**  
  - Type: Salesforce node  
  - Role: Upserts a company customer as a Salesforce Account using `External_ID__c` external ID field mapped to NetSuite internal ID.  
  - Parameters: Account Name from `companyName`, externalId from customer ID.  
  - Credentials: Salesforce OAuth2  
  - Outputs: Upsert result forwarded to merge node.  
  - Edge cases: Missing external ID field in Salesforce; Salesforce API limits/errors.

- **SF: Create or Update Person's Account**  
  - Type: Salesforce node  
  - Role: Upserts individual customer as an Account with full name from first and last names, using external ID.  
  - Credentials: Same as above.  
  - Outputs: Passes to contact upsert node.  
  - Edge cases: Data inconsistencies causing Salesforce errors.

- **SF: Create or Update Contact**  
  - Type: Salesforce node  
  - Role: Upserts Contact record associated with the Person's Account. Uses external ID, sets firstName, lastName, email, and accountId (note: accountId parameter field is misspelled as "acconuntId" in configuration and should be corrected to "accountId").  
  - Credentials: Same as above.  
  - Edge cases: Contact not linked properly if accountId is incorrect; auth or API errors.

- **Merge Company/Contact Results**  
  - Type: Merge  
  - Role: Combines upsert result streams from company and contact upserts to proceed with pagination update.  
  - Edge cases: Merge conflicts or empty inputs.

---

#### 1.6 Pagination & Loop Control

**Overview:**  
Updates pagination offset and last export date after processing each batch, then loops back if more records exist.

**Nodes Involved:**  
- Update Paging Offset and LastExportDate  
- Has More Records?

**Node Details:**

- **Update Paging Offset and LastExportDate**  
  - Type: Code  
  - Role: Increments offset by 20 (batch size), updates `hasMore` based on NetSuite response, and sets the `lastExportDate` to current date. Updates workflow static data accordingly.  
  - Outputs: Passes updated flags to the condition node.  
  - Edge cases: Incorrect date formatting or static data update failure.

- **Has More Records?**  
  - Same as in Block 1.1, reused here to decide whether to loop or finish.

---

#### 1.7 Workflow Completion

**Overview:**  
Marks the successful end of synchronization after all records are processed.

**Nodes Involved:**  
- Workflow is finished

**Node Details:**

- **Workflow is finished**  
  - Type: No Operation (NoOp)  
  - Role: Serves as the terminal node indicating workflow completion.  
  - Inputs: From Has More Records? when no more records to process.  
  - Edge cases: None.

---

### 3. Summary Table

| Node Name                       | Node Type                   | Functional Role                          | Input Node(s)                 | Output Node(s)                        | Sticky Note                                                                                                                  |
|--------------------------------|-----------------------------|----------------------------------------|------------------------------|-------------------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Execute Workflow Daily          | Schedule Trigger            | Starts workflow on daily schedule      | None                         | Init Offset                         |                                                                                                                              |
| Init Offset                    | Code                        | Initializes pagination variables       | Execute Workflow Daily        | Has More Records?                   |                                                                                                                              |
| Has More Records?              | If                          | Checks if more records to process      | Init Offset, Update Paging Offset and LastExportDate | Retrieve Paging Offset and LastExportDate, Workflow is finished |                                                                                                                              |
| Retrieve Paging Offset and LastExportDate | Code                        | Retrieves current offset and last run date | Has More Records?             | NS: Customer - Get All records      |                                                                                                                              |
| NS: Customer - Get All records | NetSuite REST API           | Retrieves batch of customers            | Retrieve Paging Offset and LastExportDate | Split Customers Array              | See sticky note about using Q parameter and NetSuite Docs: https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/section_1545222128.html |
| NS: Customer - Get Delta records | NetSuite REST API           | Retrieves batch of customers changed since last run | (Not connected in main flow) | (No outputs configured)             | Sticky Note2 explains to replace 'Get All records' node with this for delta processing                                         |
| Split Customers Array          | Split Out                   | Splits batch into individual customers | NS: Customer - Get All records | NS: Customer - Get record          |                                                                                                                              |
| NS: Customer - Get record      | NetSuite REST API           | Gets detailed customer record          | Split Customers Array         | Is Company?                        |                                                                                                                              |
| Is Company?                   | If                          | Decides if customer is company or person | NS: Customer - Get record      | SF: Create or Update Company Account (true), SF: Create or Update Person's Account (false) |                                                                                                                              |
| SF: Create or Update Company Account | Salesforce Node             | Upserts company as Salesforce Account  | Is Company? (true)            | Merge Company/Contact Results      | Sticky Note explains Salesforce section                                                                                      |
| SF: Create or Update Person's Account | Salesforce Node             | Upserts person as Salesforce Account   | Is Company? (false)           | SF: Create or Update Contact       |                                                                                                                              |
| SF: Create or Update Contact   | Salesforce Node             | Upserts contact related to person account | SF: Create or Update Person's Account | Merge Company/Contact Results      |                                                                                                                              |
| Merge Company/Contact Results  | Merge                       | Merges results from company/contact upserts | SF: Create or Update Company Account, SF: Create or Update Contact | Update Paging Offset and LastExportDate |                                                                                                                              |
| Update Paging Offset and LastExportDate | Code                        | Updates pagination offset and last export date | Merge Company/Contact Results | Has More Records?                   |                                                                                                                              |
| Workflow is finished           | NoOp                        | Marks end of workflow                   | Has More Records? (false)     | None                             |                                                                                                                              |
| Sticky Note7                  | Sticky Note                 | Workflow overview and usage instructions | None                         | None                             | Provides detailed description, usage instructions, and notes about node requirements and support contact                      |
| Sticky Note                   | Sticky Note                 | Salesforce section annotation           | None                         | None                             |                                                                                                                              |
| Sticky Note1                  | Sticky Note                 | NetSuite section annotation             | None                         | None                             |                                                                                                                              |
| Sticky Note2                  | Sticky Note                 | Explains delta records processing       | None                         | None                             |                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: "Execute Workflow Daily"  
   - Type: Schedule Trigger  
   - Configure to trigger daily (interval set to 1 day).

2. **Create Code Node to Initialize Pagination**  
   - Name: "Init Offset"  
   - Type: Code  
   - JavaScript code:  
     ```js
     const workflowStaticData = $getWorkflowStaticData('global');
     workflowStaticData.offset = 0;
     workflowStaticData.hasMore = true;
     return [{ offset: workflowStaticData.offset, hasMore: workflowStaticData.hasMore }];
     ```  
   - Connect from "Execute Workflow Daily".

3. **Create If Node to Check More Records**  
   - Name: "Has More Records?"  
   - Type: If  
   - Condition: Check if `hasMore` is true (`={{ $json.hasMore }}`)  
   - Connect from "Init Offset".

4. **Create Code Node to Retrieve Offset and LastExportDate**  
   - Name: "Retrieve Paging Offset and LastExportDate"  
   - Type: Code  
   - JavaScript:  
     ```js
     const workflowStaticData = $getWorkflowStaticData('global');
     return [{ offset: workflowStaticData.offset, lastExportDate: workflowStaticData.lastExportDate }];
     ```  
   - Connect from "Has More Records?" True branch.

5. **Create NetSuite REST Node to Get All Customers**  
   - Name: "NS: Customer - Get All records"  
   - Type: NetSuite REST  
   - Operation: GET `/customer` with parameters:  
     - limit: 20  
     - offset: `={{ $json.offset }}`  
   - Credentials: Set NetSuite REST OAuth2 credentials.  
   - Connect from "Retrieve Paging Offset and LastExportDate".

   *(Optional: For delta retrieval, replace this node with "NS: Customer - Get Delta records" node configured to query customers modified on or after `lastExportDate`.)*

6. **Create Split Out Node**  
   - Name: "Split Customers Array"  
   - Type: Split Out  
   - Field to split out: `items` (or the array field containing customers)  
   - Connect from "NS: Customer - Get All records".

7. **Create NetSuite REST Node to Get Customer Record**  
   - Name: "NS: Customer - Get record"  
   - Type: NetSuite REST  
   - Operation: GET `/customer/{id}` using `id` from split item (`={{ $json.id }}`)  
   - Credentials: Same as above.  
   - Connect from "Split Customers Array".

8. **Create If Node to Check Customer Type**  
   - Name: "Is Company?"  
   - Type: If  
   - Condition: `isPerson == false` (`={{ $json.isPerson }} == false`)  
   - Connect from "NS: Customer - Get record".

9. **Create Salesforce Node to Upsert Company Account**  
   - Name: "SF: Create or Update Company Account"  
   - Type: Salesforce  
   - Operation: Upsert Account  
   - External ID Field: `External_ID__c`  
   - External ID Value: `={{ $json.id }}`  
   - Account Name: `={{ $json.companyName }}`  
   - Credentials: Salesforce OAuth2  
   - Connect from "Is Company?" True branch.

10. **Create Salesforce Node to Upsert Person's Account**  
    - Name: "SF: Create or Update Person's Account"  
    - Type: Salesforce  
    - Operation: Upsert Account  
    - External ID Field: `External_ID__c`  
    - External ID Value: `={{ $json.id }}`  
    - Account Name: `={{ $json.firstName + ' ' + $json.lastName }}`  
    - Credentials: Salesforce OAuth2  
    - Connect from "Is Company?" False branch.

11. **Create Salesforce Node to Upsert Contact**  
    - Name: "SF: Create or Update Contact"  
    - Type: Salesforce  
    - Operation: Upsert Contact  
    - External ID Field: `External_ID__c`  
    - External ID Value: `={{ $json.id }}`  
    - Last Name: `={{ $('Is Company?').item.json.lastName }}`  
    - First Name: `={{ $('Is Company?').item.json.firstName }}`  
    - Email: `={{ $('Is Company?').item.json.email }}`  
    - AccountId: `={{ $json.id }}` (Note: Ensure parameter name is correctly set to "accountId")  
    - Credentials: Salesforce OAuth2  
    - Connect from "SF: Create or Update Person's Account".

12. **Create Merge Node**  
    - Name: "Merge Company/Contact Results"  
    - Type: Merge  
    - Mode: Wait for all inputs  
    - Connect from:  
      - "SF: Create or Update Company Account"  
      - "SF: Create or Update Contact"

13. **Create Code Node to Update Pagination**  
    - Name: "Update Paging Offset and LastExportDate"  
    - Type: Code  
    - JavaScript:  
      ```js
      const workflowStaticData = $getWorkflowStaticData('global');
      workflowStaticData.offset += 20;
      workflowStaticData.hasMore = $('NS: Customer - Get Delta records').first().json.hasMore;
      workflowStaticData.lastExportDate = new Date().toLocaleDateString('en-US');
      return [{ offset: workflowStaticData.offset, hasMore: workflowStaticData.hasMore, lastExportDate: workflowStaticData.lastExportDate }];
      ```  
    - Connect from "Merge Company/Contact Results".

14. **Connect Pagination Loop**  
    - Connect "Update Paging Offset and LastExportDate" output to "Has More Records?" node to loop or finish.

15. **Create NoOp Node for Workflow Completion**  
    - Name: "Workflow is finished"  
    - Type: No Operation (NoOp)  
    - Connect from "Has More Records?" False branch.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                        | Context or Link                                                                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| This n8n template demonstrates how to export NetSuite Customers and upsert into Salesforce, processing 20 records per iteration until completion. NetSuite Internal Id is used as the external Id in Salesforce.                                       | Sticky Note7 content                                                                                          |
| For delta synchronization, replace ‘NS: Customer - Get All records’ node with ‘NS: Customer - Get Delta records’ node and ensure the external ID field exists in Salesforce.                                                                        | Sticky Note2                                                                                                  |
| The workflow uses the NetSuite REST community node, which requires a self-hosted n8n instance.                                                                                                                                                      | Sticky Note7                                                                                                  |
| NetSuite Q parameter can filter records, e.g., email START_WITH barbara. More info at: https://docs.oracle.com/en/cloud/saas/netsuite/ns-online-help/section_1545222128.html                                                                          | Sticky Note1                                                                                                  |
| For help, contact support@entechsolutions.com                                                                                                                                                                                                       | Sticky Note7                                                                                                  |
| Important: The Salesforce "Create or Update Contact" node currently has a typo in the parameter name for AccountId ("acconuntId"). This must be corrected to "accountId" to properly link contacts to accounts.                                        | Observed configuration detail                                                                                 |

---

**Disclaimer:** The provided text is derived exclusively from an automated workflow created with n8n, respecting all applicable content policies without any illegal, offensive, or protected elements. All data processed is legal and public.