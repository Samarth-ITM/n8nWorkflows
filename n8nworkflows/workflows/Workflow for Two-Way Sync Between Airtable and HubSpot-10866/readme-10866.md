Workflow for Two-Way Sync Between Airtable and HubSpot

https://n8nworkflows.xyz/workflows/workflow-for-two-way-sync-between-airtable-and-hubspot-10866


# Workflow for Two-Way Sync Between Airtable and HubSpot

### 1. Workflow Overview

This workflow automates a robust two-way synchronization process between Airtable (acting as a lead control panel) and HubSpot CRM, targeting sales teams or marketing operations managing lead data. It is designed as a "Human-in-the-Loop" system where leads are manually approved in Airtable before being synced to HubSpot. The workflow ensures reliable batch processing, error handling, and data consistency, supporting incremental syncing and preventing duplication or race conditions.

**Logical Blocks:**

- **1.1 Scheduled Polling & Batch Processing:** Periodically triggers the workflow to fetch batches of approved leads from Airtable and process them one by one.
- **1.2 Record Locking & Preparation:** Locks each record in Airtable to prevent concurrent processing and extracts the email domain for company lookup.
- **1.3 Company Synchronization:** Searches HubSpot for a company by domain; creates the company if it does not exist.
- **1.4 Contact Synchronization:** Creates or updates a contact in HubSpot, associates it with the company.
- **1.5 Airtable Sync-Back & Status Updates:** Updates Airtable with HubSpot IDs and sync status; handles success and failure reporting.
- **1.6 Error Handling & Logging:** Captures and logs errors back into Airtable, allowing the workflow to continue processing subsequent leads.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Polling & Batch Processing

- **Overview:** This block triggers the workflow every 20 seconds (configurable), retrieves up to 50 leads marked "👍 Ready to Sync" from Airtable, and processes them individually in a loop.
- **Nodes Involved:** `Schedule Trigger`, `get 👍Ready to Sync`, `Loop Over Items`, `🎉All Records Completed!🎉`
  
**Node Details:**

- **Schedule Trigger**
  - Type: Schedule Trigger
  - Role: Initiates workflow on a fixed interval (every 20 seconds)
  - Configuration: Interval set to 20 seconds
  - Input/Output: None input; outputs to `get 👍Ready to Sync`
  - Edge Cases: If the interval is too short, simultaneous executions can overlap causing race conditions.

- **get 👍Ready to Sync**
  - Type: Airtable Node (Search operation)
  - Role: Fetches up to 50 lead records from Airtable view `👍 Ready to Sync`
  - Configuration: Searches a specific Airtable base and table; limits to 50 records per run
  - Input: Trigger from Schedule Trigger
  - Output: List of lead records for batch processing
  - Credentials: Airtable Personal Access Token
  - Edge Cases: API rate limits; empty result sets; incorrect base/view configuration

- **Loop Over Items**
  - Type: SplitInBatches
  - Role: Processes each lead individually, outputting one item at a time to downstream nodes
  - Configuration: Default batch size = 1 (process one record at a time)
  - Input: Output of Airtable search
  - Output: Two outputs: (1) Next Item (current record), (2) No Items (empty batch)
  - Edge Cases: If no items, triggers No Items output to end the cycle

- **🎉All Records Completed!🎉**
  - Type: NoOp (No Operation)
  - Role: Marks the end of batch processing when no more leads are found
  - Input: No Items output from Loop Over Items
  - Output: None
  - Edge Cases: None

---

#### 2.2 Record Locking & Preparation

- **Overview:** Locks the current lead record by updating its status to "🔄 Syncing..." to prevent reprocessing, then extracts the domain from the lead's email to assist company search in HubSpot.
- **Nodes Involved:** `change status to Syncing...`, `get domain`

**Node Details:**

- **change status to Syncing...**
  - Type: Airtable Node (Update operation)
  - Role: Updates Airtable record status to "🔄 Syncing..."
  - Configuration: Updates current record by ID, changes "Status" field
  - Input: Current lead from Loop Over Items
  - Output: Updated record passed to `get domain`
  - Credentials: Airtable Personal Access Token
  - Edge Cases: Airtable API failures, concurrent updates causing conflicts

- **get domain**
  - Type: Set Node
  - Role: Extracts domain part from the lead's email address (e.g. from user@company.com extracts company.com)
  - Configuration: Uses expression `={{ $json.fields.Email.split('@')[1] }}`
  - Input: Updated record from `change status to Syncing...`
  - Output: JSON containing extracted domain for company lookup
  - Edge Cases: Malformed email addresses causing undefined or errors in split

---

#### 2.3 Company Synchronization

- **Overview:** Searches HubSpot for a company matching the extracted email domain. If none is found, creates a new company record.
- **Nodes Involved:** `Search company`, `If company exists`, `Create a company`, `Merge`

**Node Details:**

- **Search company**
  - Type: HubSpot Node (Search Company by Domain)
  - Role: Queries HubSpot CRM for company with domain extracted earlier
  - Configuration: Domain parameter set as `=https://{{ $json.domain }}`
  - Input: Domain from `get domain`
  - Output: Company object if found; empty if not
  - Credentials: HubSpot App Token
  - Edge Cases: API rate limits, invalid domain format, authentication failures

- **If company exists**
  - Type: If Node
  - Role: Checks if companyId field exists (indicating company found)
  - Configuration: Condition tests if `companyId` is not empty
  - Input: Output of `Search company`
  - Output: True (company exists) to `Merge`, False to `Create a company`
  - Edge Cases: Missing or malformed companyId causing logic errors

- **Create a company**
  - Type: HubSpot Node (Create Company)
  - Role: Creates new company in HubSpot with company name and domain
  - Configuration: Uses company name from Airtable, domain from `get domain` for website and companyDomainName fields
  - Input: False path from `If company exists`
  - Output: Created company object to `Merge`
  - OnError: Continue on error to not stop entire workflow
  - Credentials: HubSpot App Token
  - Edge Cases: API errors, duplicate companies, malformed inputs

- **Merge**
  - Type: Merge Node
  - Role: Merges the true (existing company) and false (newly created company) paths into a single stream with a valid `companyId`
  - Configuration: Default merge mode
  - Input: Two inputs from `If company exists` and `Create a company`
  - Output: Unified company object for next steps
  - Edge Cases: Merge conflicts or empty data causing downstream failures

---

#### 2.4 Contact Synchronization

- **Overview:** Creates or updates a contact in HubSpot using the lead’s email as unique key and associates the contact with the identified company.
- **Nodes Involved:** `Create or update a contact`

**Node Details:**

- **Create or update a contact**
  - Type: HubSpot Node (Create or Update Contact)
  - Role: Creates or updates contact record with lead details and associates it with company ID from Merge
  - Configuration:
    - Email: from Airtable lead
    - Additional fields: jobTitle, firstName, associatedCompanyId (from Merge)
    - resolveData option disabled for performance
  - Input: Output of `Merge`
  - Output: Contact object with unique HubSpot contact ID (vid)
  - OnError: Continue on error, to allow error handling downstream
  - Credentials: HubSpot App Token
  - Edge Cases: Duplicate contacts, API failures, missing email or companyId

---

#### 2.5 Airtable Sync-Back & Status Updates

- **Overview:** Updates Airtable lead record with HubSpot company and contact IDs, sets status to "✅ Synced", or logs failure status and message.
- **Nodes Involved:** `Sync back`, `Report Failure`, `👍Done! Going for next record`, `👎Failed! Going for next record1`

**Node Details:**

- **Sync back**
  - Type: Airtable Node (Update operation)
  - Role: Writes back HubSpot IDs and success status to Airtable, adds a note with timestamp
  - Configuration: Updates record by ID, fields include Status, Note, HubSpot Contact ID, HubSpot Company ID
  - Input: Output of `Create or update a contact`
  - Output: Success flow to `👍Done! Going for next record`
  - Credentials: Airtable Personal Access Token
  - Edge Cases: Airtable update failures, data type mismatches

- **Report Failure**
  - Type: Airtable Node (Update operation)
  - Role: Logs error details back to Airtable row, sets status to "❌ Error", clears HubSpot IDs
  - Configuration: Updates record by ID, sets Note with error JSON and timestamp
  - Input: Error outputs from `Create a company`, `Create or update a contact`, `Merge`, `Sync back`
  - Output: Failure flow to `👎Failed! Going for next record1`
  - Credentials: Airtable Personal Access Token
  - Edge Cases: Complex or large error messages truncation, API failures

- **👍Done! Going for next record**
  - Type: NoOp
  - Role: Marks successful processing, loops back to `Loop Over Items` for next lead
  - Input: Success output from `Sync back`
  - Output: Back to `Loop Over Items`
  
- **👎Failed! Going for next record1**
  - Type: NoOp
  - Role: Marks failure handling done, loops back to `Loop Over Items` for next lead
  - Input: Output from `Report Failure`
  - Output: Back to `Loop Over Items`

---

### 3. Summary Table

| Node Name                     | Node Type         | Functional Role                              | Input Node(s)                  | Output Node(s)                             | Sticky Note                                                                                              |
|-------------------------------|-------------------|----------------------------------------------|-------------------------------|--------------------------------------------|---------------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger  | Start workflow on timer                       | None                          | get 👍Ready to Sync                        | Step 1: Set Your Schedule (Polling trigger every 20 sec; recommended to adjust as needed)                |
| get 👍Ready to Sync           | Airtable          | Fetch batch of leads from Airtable            | Schedule Trigger              | Loop Over Items                            | Step 2: Configure Airtable (Fetch up to 50 leads from "👍 Ready to Sync" view)                            |
| Loop Over Items              | SplitInBatches    | Process each lead record individually          | get 👍Ready to Sync           | change status to Syncing..., 🎉All Records Completed!🎉 | Logic: The Batch Engine (Processes one item at a time, outputs No Items or Next Item)                     |
| 🎉All Records Completed!🎉   | NoOp              | Marks end of batch processing                  | Loop Over Items (No Items)    | None                                      | All Done! (Ends poll cycle if no more records)                                                          |
| change status to Syncing...   | Airtable          | Lock record by setting status to "Syncing..."| Loop Over Items (Next Item)   | get domain                                | Logic: Lock the Record (Prevents re-processing during sync)                                             |
| get domain                   | Set               | Extract email domain for company lookup        | change status to Syncing...   | Search company                            | Logic: Helper Node (Extracts domain from email)                                                          |
| Search company              | HubSpot           | Search HubSpot for company by domain           | get domain                   | If company exists                         | Step 3: Configure HubSpot (Search for company by domain)                                                |
| If company exists            | If                | Check if company exists or not                  | Search company               | Merge (true), Create a company (false)   | Logic: Check if Company Was Found (Branches on existence)                                               |
| Create a company             | HubSpot           | Create new company in HubSpot                    | If company exists (false)    | Merge, Report Failure                     | Logic: Create company if not found                                                                      |
| Merge                       | Merge             | Combine found and created company paths         | If company exists (true), Create a company | Create or update a contact, Report Failure | Logic: Combine Paths (Unify company data)                                                               |
| Create or update a contact   | HubSpot           | Create or update contact and associate with company | Merge                       | Sync back, Report Failure                  | Logic: Create/Update Contact (Uses email as unique key)                                                 |
| Sync back                   | Airtable          | Update Airtable record with HubSpot IDs and success status | Create or update a contact  | 👍Done! Going for next record, Report Failure | Success: Sync to Airtable (Final feedback loop)                                                        |
| Report Failure              | Airtable          | Log error details in Airtable, mark error status | Multiple nodes on error      | 👎Failed! Going for next record1           | Error Handling (Logs failure, continues processing)                                                    |
| 👍Done! Going for next record | NoOp              | Marks successful processing and loops back      | Sync back                    | Loop Over Items                            |                                                                                                         |
| 👎Failed! Going for next record1| NoOp            | Marks failure handling and loops back           | Report Failure               | Loop Over Items                            |                                                                                                         |
| Sticky Note (various)       | Sticky Note       | Documentation and guidance                       | None                        | None                                      | Multiple sticky notes provide step instructions, explanations, and links to resources                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**
   - Type: Schedule Trigger
   - Set interval to run every 20 seconds (adjustable)
   - Connect output to next node

2. **Create Airtable node to fetch leads**
   - Type: Airtable (Search operation)
   - Configure Airtable credential (Personal Access Token)
   - Select your Airtable base and table matching the "Sales Lead Pipeline"
   - Set view to `👍 Ready to Sync`
   - Limit results to 50
   - Connect input from Schedule Trigger

3. **Create SplitInBatches node**
   - Type: SplitInBatches
   - Leave default batch size (1)
   - Connect input from Airtable fetch node

4. **Create Airtable node to update record status ("🔄 Syncing...")**
   - Type: Airtable (Update operation)
   - Configure Airtable credential
   - Select same base and table
   - Map record ID from current item
   - Update `Status` field to "🔄 Syncing..."
   - Connect input from SplitInBatches output for "Next Item"

5. **Create Set node to extract domain**
   - Type: Set
   - Add new field `domain` with expression: `={{ $json.fields.Email.split('@')[1] }}`
   - Input from Airtable update node

6. **Create HubSpot node to search company**
   - Type: HubSpot (Search Company by Domain)
   - Authenticate with HubSpot App Token
   - Set domain parameter as `=https://{{ $json.domain }}`
   - Input from Set node

7. **Create If node to check company existence**
   - Type: If
   - Condition: Check if `companyId` property is not empty in HubSpot search output
   - True path to merge node
   - False path to create company node

8. **Create HubSpot node to create company**
   - Type: HubSpot (Create Company)
   - Set company name from Airtable lead field `Company Name`
   - Set websiteUrl and companyDomainName from extracted domain
   - Enable "Continue on error" option
   - Input from If node False output

9. **Create Merge node**
   - Type: Merge
   - Merge True input from If node True output (existing company)
   - Merge False input from Create company node output
   - Output combined company data

10. **Create HubSpot node to create/update contact**
    - Type: HubSpot (Create or Update Contact)
    - Email from Airtable lead `Email`
    - Additional fields: `firstName` (Name), `jobTitle` (Job Title), `associatedCompanyId` (from Merge output `companyId`)
    - Disable resolveData option for performance
    - Enable continue on error
    - Input from Merge node

11. **Create Airtable node to sync back success**
    - Type: Airtable (Update operation)
    - Update record by ID
    - Fields: `Status` to "✅ Synced", `Note` with success timestamp, `HubSpot Contact ID`, `HubSpot Company ID`
    - Input from HubSpot contact node success output

12. **Create Airtable node to report failure**
    - Type: Airtable (Update operation)
    - Update record by ID
    - Fields: `Status` to "❌ Error", `Note` with error message and timestamp, clear HubSpot IDs
    - Input from error outputs of Create company, Merge, Create/update contact, and Sync back nodes

13. **Create NoOp node for success loop continuation**
    - Connect input from Airtable sync back success output
    - Output connects back to SplitInBatches node to process next item

14. **Create NoOp node for failure loop continuation**
    - Connect input from Airtable report failure node output
    - Output connects back to SplitInBatches node to process next item

15. **Create NoOp node for batch completion**
    - Connect input from SplitInBatches node No Items output
    - Marks end of current poll cycle

16. **Add sticky notes (optional)**
    - Add detailed notes explaining each step, setup instructions, and links to resources

---

### 5. General Notes & Resources

| Note Content                                                                                                                                          | Context or Link                                                                                                   |
|-------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This template solves a common business problem: syncing leads from Airtable to HubSpot in a robust, error-proof way.                                 | Sticky Note "🚀 Welcome!"                                                                                         |
| Airtable Base Template: [Click Here to Copy the Airtable Base](https://airtable.com/appthZgrzxKzQPEYa/shrRoRtIXoL1tlDVv)                             | Airtable base structure required for this workflow                                                               |
| Video Guide: How to Connect Airtable to n8n                                                                                                          | https://www.youtube.com/watch?v=v_xFFfkBeI2rQ                                                                    |
| Video Guide: How to Connect HubSpot to n8n (Private Apps)                                                                                            | https://www.youtube.com/watch?v=v_KzP52kRsRrk                                                                    |
| Use a HubSpot Developer Sandbox for safe testing                                                                                                    | Recommended for risk-free environment                                                                             |
| Polling interval is set to 20 seconds for testing; adjust to 5 minutes or 1 hour for production                                                       | Sticky Note on Schedule Trigger                                                                                   |
| This workflow implements professional safeguards such as locking records during sync to avoid duplicates or race conditions                         | Sticky Note on "Lock the Record"                                                                                   |
| Error handling is per-item; errors do not stop the entire workflow but get logged back in Airtable for review                                       | Sticky Note on Error Handling                                                                                       |
| HubSpot Private App credentials are used for authentication; ensure they have appropriate permissions                                                | Credential note                                                                                                    |

---

This documentation provides a complete understanding of the workflow, enabling advanced users or AI agents to reproduce, modify, and maintain the system reliably.