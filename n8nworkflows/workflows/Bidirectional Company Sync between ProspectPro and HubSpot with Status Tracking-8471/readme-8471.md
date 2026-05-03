Bidirectional Company Sync between ProspectPro and HubSpot with Status Tracking

https://n8nworkflows.xyz/workflows/bidirectional-company-sync-between-prospectpro-and-hubspot-with-status-tracking-8471


# Bidirectional Company Sync between ProspectPro and HubSpot with Status Tracking

---

### 1. Workflow Overview

This workflow titled **"Bidirectional Company Sync between ProspectPro and HubSpot with Status Tracking"** is designed to synchronize company (prospect) data from ProspectPro into HubSpot, ensuring both platforms are aligned. The primary use case is to keep HubSpot’s company records updated or created based on ProspectPro data, with clear status tracking through tagging in ProspectPro to mark success or failure of synchronization attempts.

The workflow is structured into these logical blocks:

- **1.1 Input Reception & Validation:** Receives a ProspectPro ID, retrieves the prospect, and verifies data integrity and absence of previous sync errors.
- **1.2 HubSpot Company Search:** Attempts to find the corresponding company in HubSpot using the ProspectPro ID or the company domain.
- **1.3 HubSpot Company Update or Creation:** Updates existing HubSpot company records or creates new ones based on ProspectPro data.
- **1.4 Sync Status Logging:** Updates ProspectPro tags to reflect whether the sync succeeded or failed.
- **1.5 Sub-Workflow and Trigger Setup:** Supports invocation from other workflows or direct triggering for automated runs.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Validation

- **Overview:**  
Starts the workflow by receiving a ProspectPro company ID, retrieving detailed prospect data, and verifying whether previous sync errors exist. It ensures only valid and error-free prospects proceed to syncing.

- **Nodes Involved:**  
  - When Executed by Another Workflow  
  - Get prospect  
  - No Existing Errors?  
  - Continue?  
  - return { result: false };

- **Node Details:**

  - **When Executed by Another Workflow**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Entry point for external invocation, expects input containing the ProspectPro company ID.  
    - *Config:* Input parameter named `id` captured from calling workflow.  
    - *Connections:* Outputs to `Get prospect`.  
    - *Failure Modes:* Missing `id` input results in no data to proceed.  
   
  - **Get prospect**  
    - *Type:* ProspectPro node (API call)  
    - *Role:* Retrieves detailed prospect data from ProspectPro by ID.  
    - *Config:* Uses the incoming `id` field for lookup, credentials set to ProspectPro API.  
    - *Error Handling:* On error, continues with error output to prevent workflow crash.  
    - *Connections:* Outputs to `No Existing Errors?`.  
    - *Edge Cases:* API rate limits, invalid ID, or ProspectPro service outages.  
   
  - **No Existing Errors?**  
    - *Type:* If node  
    - *Role:* Checks if the prospect's tags include `"HubspotSyncFailed"`.  
    - *Config:* Condition: tags do not contain `"HubspotSyncFailed"`.  
    - *Connections:* If no error tag, proceeds to `Continue?`; else outputs to `return { result: false }` to quit workflow.  
    - *Edge Cases:* Tags missing or in unexpected format.  
   
  - **Continue?**  
    - *Type:* If node  
    - *Role:* Determines if syncing should continue based on presence of key fields like domain and id.  
    - *Config:* Checks if `domain` and `id` are present and non-empty in prospect data.  
    - *Connections:* If true, proceeds to `Search Companies by Bedrijfsdata ID`; else sets failure tag later in the flow.  
    - *Edge Cases:* Missing domain or id fields prevent sync continuation.  
   
  - **return { result: false };**  
    - *Type:* Code node  
    - *Role:* Ends workflow with a false result flag indicating no sync occurred due to validation failure or errors.  
    - *Connections:* Terminal node for failure branches.

#### 2.2 HubSpot Company Search

- **Overview:**  
Searches HubSpot for an existing company matching either the ProspectPro ID or the company domain to decide whether to update or create a HubSpot company record.

- **Nodes Involved:**  
  - Search Companies by Bedrijfsdata ID  
  - Found by ID?  
  - Has Domain?  
  - Search Companies by Domain  
  - Found by Domain?  
  - Set hsCompany

- **Node Details:**

  - **Search Companies by Bedrijfsdata ID**  
    - *Type:* HTTP Request node  
    - *Role:* Makes a POST request to HubSpot's API to find companies with a matching `prospectpro_id`.  
    - *Config:* Uses HubSpot OAuth2 credentials; queries properties including prospectpro_id, domain, name, lifecyclestage.  
    - *Connections:* If found, goes to `Set hsCompany`; else to `Set Tag: HubspotSyncFailed`.  
    - *Edge Cases:* API rate limits, authentication failures, empty results.  
   
  - **Found by ID?**  
    - *Type:* If node  
    - *Role:* Checks if HubSpot search by ID returned results (total > 0).  
    - *Connections:* True → `Set hsCompany`; False → `Has Domain?`.  
   
  - **Has Domain?**  
    - *Type:* If node  
    - *Role:* Checks if the prospect has a domain to use for HubSpot search fallback.  
    - *Connections:* True → `Search Companies by Domain`; False → `Create a company`.  
   
  - **Search Companies by Domain**  
    - *Type:* HTTP Request node  
    - *Role:* Searches HubSpot companies by domain property as a fallback if no match by ID.  
    - *Config:* Queries company properties with domain filter equal to prospect domain.  
    - *Connections:* True → `Found by Domain?`.  
    - *Edge Cases:* Domain may be missing or ambiguous.  
   
  - **Found by Domain?**  
    - *Type:* If node  
    - *Role:* Checks if search by domain found any companies.  
    - *Connections:* True → `Set hsCompany`; False → `Create a company`.  
   
  - **Set hsCompany**  
    - *Type:* Code node  
    - *Role:* Extracts the first company found from search results to use for update.  
    - *Connections:* Proceeds to `Update a company`.  
    - *Edge Cases:* Empty results, unexpected data shape.

#### 2.3 HubSpot Company Update or Creation

- **Overview:**  
Depending on search results, updates an existing HubSpot company with ProspectPro data or creates a new HubSpot company record.

- **Nodes Involved:**  
  - Update a company  
  - Update Successful?  
  - Create a company  
  - Creation Successful?

- **Node Details:**

  - **Update a company**  
    - *Type:* HubSpot node  
    - *Role:* Updates company properties in HubSpot.  
    - *Config:* Uses company ID from `Set hsCompany`; updates domain, tags, and custom properties like `prospectpro_id` and `kvk`.  
    - *Error Handling:* Continues on error to allow tagging failure.  
    - *Connections:* Proceeds to `Update Successful?`.  
    - *Edge Cases:* API errors, authentication failure, invalid company ID.  
   
  - **Update Successful?**  
    - *Type:* If node  
    - *Role:* Checks if update returned a valid `companyId`, indicating success.  
    - *Connections:* Success → `Set Tag: HubspotSynced`; Failure → `Set Tag: HubspotSyncFailed`.  
   
  - **Create a company**  
    - *Type:* HubSpot node  
    - *Role:* Creates new company record in HubSpot with ProspectPro details.  
    - *Config:* Sets multiple fields including city, postal code, domain, revenue, employees, LinkedIn, and custom properties.  
    - *Error Handling:* Continues on error.  
    - *Connections:* Proceeds to `Creation Successful?`.  
    - *Edge Cases:* API limits, missing mandatory fields, invalid data formats.  
   
  - **Creation Successful?**  
    - *Type:* If node  
    - *Role:* Checks if creation returned a valid `companyId`.  
    - *Connections:* Success → `Set Tag: HubspotSynced`; Failure → `Set Tag: HubspotSyncFailed`.

#### 2.4 Sync Status Logging

- **Overview:**  
Updates ProspectPro tags to indicate whether the sync process succeeded or failed, enabling easy status tracking and error detection.

- **Nodes Involved:**  
  - Set Tag: HubspotSynced  
  - Set Tag: HubspotSyncFailed  
  - Update Tags: Success  
  - Update Tags: Fail  
  - return { result: true };  
  - return { result: false };

- **Node Details:**

  - **Set Tag: HubspotSynced**  
    - *Type:* Code node  
    - *Role:* Adds `"HubspotSynced"` tag to the ProspectPro tags array.  
    - *Connections:* Proceeds to `Update Tags: Success`.  
    - *Edge Cases:* Tags field missing or malformed.  
   
  - **Set Tag: HubspotSyncFailed**  
    - *Type:* Code node  
    - *Role:* Adds `"HubspotSyncFailed"` tag to ProspectPro tags.  
    - *Connections:* Proceeds to `Update Tags: Fail`.  
   
  - **Update Tags: Success**  
    - *Type:* ProspectPro node (patch operation)  
    - *Role:* Updates ProspectPro prospect tags with the new success tag.  
    - *Connections:* Ends with `return { result: true }`.  
    - *Edge Cases:* API failure, permission issues.  
   
  - **Update Tags: Fail**  
    - *Type:* ProspectPro node (patch operation)  
    - *Role:* Updates ProspectPro prospect tags with failure tag.  
    - *Connections:* Ends with `return { result: false }`.  
   
  - **return { result: true };**  
    - *Type:* Code node  
    - *Role:* Signals successful sync completion.  
   
  - **return { result: false };**  
    - *Type:* Code node  
    - *Role:* Signals sync failure or early exit.

#### 2.5 Sub-Workflow and Trigger Setup

- **Overview:**  
Supports running as a sub-workflow triggered by other workflows or as a standalone triggered workflow via ProspectPro trigger.

- **Nodes Involved:**  
  - ProspectPro Trigger Example  
  - When Executed by Another Workflow

- **Node Details:**

  - **ProspectPro Trigger Example**  
    - *Type:* ProspectPro Trigger node  
    - *Role:* Polls ProspectPro every minute for new prospects that do not have the `"HubspotSyncFailed"` tag to trigger this sync workflow.  
    - *Config:* Poll mode set to every minute, excludes prospects tagged with `"HubspotSyncFailed"`.  
    - *Edge Cases:* Polling delay, API rate limits.  
   
  - **When Executed by Another Workflow**  
    - As detailed in 2.1, allows external workflows to trigger this sync flow with a ProspectPro ID.

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                         | Input Node(s)                       | Output Node(s)                            | Sticky Note                                                                                                      |
|--------------------------------|----------------------------------|---------------------------------------|-----------------------------------|------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger          | Entry point, receives ProspectPro ID  | -                                 | Get prospect                             |                                                                                                                  |
| Get prospect                   | ProspectPro API Node              | Retrieve prospect details by ID       | When Executed by Another Workflow | No Existing Errors?                      | Start: Assumes input ProspectPro ID. Check errors to quit early. Set your own sync conditions in Continue? node. |
| No Existing Errors?            | If Node                         | Verify no previous sync errors exist  | Get prospect                      | Continue? / return { result: false }     |                                                                                                                  |
| Continue?                     | If Node                         | Check if essential data exists to sync | No Existing Errors?               | Search Companies by Bedrijfsdata ID / Set Tag: HubspotSyncFailed |                                                                                                                  |
| Search Companies by Bedrijfsdata ID | HTTP Request                    | Search HubSpot companies by ProspectPro ID | Continue?                     | Found by ID?                            | Search & Select HubSpot Company: Use HTTP nodes for search due to HubSpot node limitations.                       |
| Found by ID?                  | If Node                         | Check if company found by ID           | Search Companies by Bedrijfsdata ID | Set hsCompany / Has Domain?              |                                                                                                                  |
| Has Domain?                   | If Node                         | Check if prospect has domain            | Found by ID?                     | Search Companies by Domain / Create a company |                                                                                                                  |
| Search Companies by Domain    | HTTP Request                    | Search HubSpot companies by domain     | Has Domain?                     | Found by Domain?                         |                                                                                                                  |
| Found by Domain?              | If Node                         | Check if company found by domain       | Search Companies by Domain       | Set hsCompany / Create a company          |                                                                                                                  |
| Set hsCompany                 | Code Node                      | Extract first HubSpot company from search | Found by ID? / Found by Domain? | Update a company                          |                                                                                                                  |
| Update a company              | HubSpot Node                   | Update existing HubSpot company         | Set hsCompany                   | Update Successful?                       | Update HubSpot Company: Place to also update ProspectPro from HubSpot if desired.                                |
| Update Successful?            | If Node                         | Verify update success                    | Update a company                | Set Tag: HubspotSynced / Set Tag: HubspotSyncFailed |                                                                                                                  |
| Create a company              | HubSpot Node                   | Create new HubSpot company               | Has Domain? / Found by Domain? | Creation Successful?                      | Create HubSpot Company: Creates companies not found in HubSpot.                                                 |
| Creation Successful?          | If Node                         | Verify creation success                   | Create a company               | Set Tag: HubspotSynced / Set Tag: HubspotSyncFailed |                                                                                                                  |
| Set Tag: HubspotSynced        | Code Node                      | Add "HubspotSynced" tag to prospect     | Update Successful? / Creation Successful? | Update Tags: Success                  | Log Sync status to ProspectPro: Add tag to indicate successful sync.                                            |
| Update Tags: Success          | ProspectPro Patch Node          | Update ProspectPro tags with success    | Set Tag: HubspotSynced          | return { result: true }                   |                                                                                                                  |
| Set Tag: HubspotSyncFailed    | Code Node                      | Add "HubspotSyncFailed" tag to prospect | Continue? (fail) / Update Successful? (fail) / Creation Successful? (fail) / No Existing Errors? (fail) | Update Tags: Fail              | Log Company Errors: Tag prospect for manual issue resolution.                                                   |
| Update Tags: Fail             | ProspectPro Patch Node          | Update ProspectPro tags with failure    | Set Tag: HubspotSyncFailed       | return { result: false }                  |                                                                                                                  |
| return { result: true };      | Code Node                      | Signal successful workflow completion    | Update Tags: Success             | -                                        | Set output: Workflow outputs `{ result: Boolean }`.                                                              |
| return { result: false };     | Code Node                      | Signal unsuccessful or early exit        | Various failure nodes           | -                                        |                                                                                                                  |
| ProspectPro Trigger Example   | ProspectPro Trigger             | Trigger workflow on new prospects without failure tag | -                             | When Executed by Another Workflow (via external linkage) | Sync Prospects to Hubspot: Describes usage, tags, and contact info for support.                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**  
   - Add a **ProspectPro Trigger** node named `ProspectPro Trigger Example`.  
   - Configure to poll every minute, exclude prospects tagged `"HubspotSyncFailed"`.  
   - Connect its output to **Execute Workflow Trigger** node if this workflow is used as a sub-workflow.

2. **Create Entry Point Node:**  
   - Add an **Execute Workflow Trigger** node named `When Executed by Another Workflow`.  
   - Configure to accept input parameter: `id` (ProspectPro company ID).  
   - Connect output to next node.

3. **Retrieve Prospect Data:**  
   - Add **ProspectPro node** named `Get prospect`.  
   - Set operation to `get`.  
   - Parameter `id` set to `={{ $json.id }}` to use input ID.  
   - Configure API credentials for ProspectPro.  
   - On error, set "Continue On Error" to continue gracefully.  
   - Connect output to next node.

4. **Check for Previous Errors:**  
   - Add an **If node** named `No Existing Errors?`.  
   - Condition: Check if prospect tags do NOT include `"HubspotSyncFailed"`. Use expression `={{ !$json.tags.includes("HubspotSyncFailed") }}`.  
   - True path continues; False path connects to a **Code node** `return { result: false };` to terminate early.

5. **Check Required Data for Sync:**  
   - Add an **If node** named `Continue?`.  
   - Conditions:  
     - `domain` exists and is non-empty: `={{ !!$json.domain }}`  
     - `id` exists and is non-empty: `={{ !!$json.id }}`  
   - True path proceeds to HubSpot search; False path to failure tag.

6. **Search HubSpot Companies by ProspectPro ID:**  
   - Add **HTTP Request** node `Search Companies by Bedrijfsdata ID`.  
   - POST request to `https://api.hubapi.com/crm/v3/objects/companies/search`.  
   - JSON body filters on `prospectpro_id` equal to `{{$json.id}}`.  
   - Use HubSpot OAuth2 credentials.  
   - Connect output to `Found by ID?`.

7. **If Company Found by ID:**  
   - Add **If node** `Found by ID?`.  
   - Condition: Check if response `total` property > 0.  
   - True path to `Set hsCompany`.  
   - False path to `Has Domain?`.

8. **Check if Prospect Has Domain:**  
   - Add **If node** `Has Domain?`.  
   - Condition: Check if `domain` exists and is non-empty.  
   - True path to `Search Companies by Domain`.  
   - False path to `Create a company`.

9. **Search HubSpot Companies by Domain:**  
   - Add **HTTP Request** node `Search Companies by Domain`.  
   - POST to HubSpot companies search endpoint with filter on `domain` equal to prospect domain.  
   - Use HubSpot OAuth2 credentials.  
   - Connect output to `Found by Domain?`.

10. **If Company Found by Domain:**  
    - Add **If node** `Found by Domain?`.  
    - Condition: Check if `total` > 0.  
    - True path to `Set hsCompany`.  
    - False path to `Create a company`.

11. **Extract HubSpot Company Data:**  
    - Add **Code node** `Set hsCompany`.  
    - Code: `const hsCompany = $json.results[0]; return { ...hsCompany };`  
    - Connect to `Update a company`.

12. **Update Existing HubSpot Company:**  
    - Add **HubSpot node** `Update a company`.  
    - Operation: Update company by ID using `{{$json.id}}`.  
    - Update fields with ProspectPro data: domain, custom properties `prospectpro_id`, `kvk` etc.  
    - Use HubSpot OAuth2 credentials.  
    - On error, continue workflow.  
    - Connect output to `Update Successful?`.

13. **Check Update Success:**  
    - Add **If node** `Update Successful?`.  
    - Condition: Check if `companyId` is present.  
    - True path to `Set Tag: HubspotSynced`.  
    - False path to `Set Tag: HubspotSyncFailed`.

14. **Create New HubSpot Company:**  
    - Add **HubSpot node** `Create a company`.  
    - Operation: Create company with ProspectPro data (name, city, postal code, domain, phone, revenue, employees, LinkedIn, custom properties).  
    - Use HubSpot OAuth2 credentials.  
    - On error, continue workflow.  
    - Connect output to `Creation Successful?`.

15. **Check Creation Success:**  
    - Add **If node** `Creation Successful?`.  
    - Condition: Check if `companyId` exists.  
    - True path to `Set Tag: HubspotSynced`.  
    - False path to `Set Tag: HubspotSyncFailed`.

16. **Set Success Tag on Prospect:**  
    - Add **Code node** `Set Tag: HubspotSynced`.  
    - Code: Append `"HubspotSynced"` to `tags` array (splitting and joining with pipe delimiter as needed).  
    - Connect to `Update Tags: Success`.

17. **Set Failure Tag on Prospect:**  
    - Add **Code node** `Set Tag: HubspotSyncFailed`.  
    - Code: Append `"HubspotSyncFailed"` tag similarly.  
    - Connect to `Update Tags: Fail`.

18. **Update ProspectPro Tags for Success:**  
    - Add **ProspectPro node** `Update Tags: Success`.  
    - Operation: Patch prospect by ID, update `tags` field with new tags including success tag.  
    - Use ProspectPro API credentials.  
    - On success connect to `return { result: true };`.

19. **Update ProspectPro Tags for Failure:**  
    - Add **ProspectPro node** `Update Tags: Fail`.  
    - Operation: Patch prospect tags including failure tag.  
    - Connect to `return { result: false };`.

20. **Return Success Result:**  
    - Add **Code node** `return { result: true };`.  
    - Returns JSON with `result: true`.  
    - Terminal node.

21. **Return Failure Result:**  
    - Add **Code node** `return { result: false };`.  
    - Returns JSON with `result: false`.  
    - Terminal node.

22. **Optional Sticky Notes:**  
    - Add sticky notes with documented explanations at key points for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow synchronizes prospects from ProspectPro to HubSpot and uses tags to track sync status.                                                                                                                                          | Main workflow description                                                                        |
| Use HTTP Request nodes for HubSpot company search because the native HubSpot node lacks search capabilities.                                                                                                                                  | Sticky note near "Search & Select Hubspot Company" block                                         |
| When a sync fails, the prospect is tagged `"HubspotSyncFailed"` to prevent repeated failed syncs until manually resolved.                                                                                                                    | Tags usage across the workflow                                                                  |
| For support and best practice tips, contact ProspectPro customer service at https://www.prospectpro.nl/klantenservice/.                                                                                                                       | Sticky note in workflow start                                                                   |
| Workflow outputs a simple JSON `{ result: Boolean }` indicating success or failure, useful for integration or chaining in other workflows.                                                                                                    | Sticky note near output code nodes                                                              |
| Polling interval in ProspectPro trigger is set to one minute; adjust carefully to avoid API rate limits.                                                                                                                                       | Trigger node configuration notes                                                                |
| Use OAuth2 credentials for HubSpot API calls; ensure tokens are valid and refreshed as needed.                                                                                                                                                  | Credentials setup for HubSpot nodes                                                             |
| Handle errors gracefully by setting tags and continuing workflow to avoid crashes or duplicated runs without manual intervention.                                                                                                            | Error handling strategy                                                                          |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow built with n8n, an integration and automation tool. The processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.

---