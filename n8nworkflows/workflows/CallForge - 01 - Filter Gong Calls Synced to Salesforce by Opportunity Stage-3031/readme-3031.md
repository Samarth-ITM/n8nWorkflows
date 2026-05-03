CallForge - 01 - Filter Gong Calls Synced to Salesforce by Opportunity Stage

https://n8nworkflows.xyz/workflows/callforge---01---filter-gong-calls-synced-to-salesforce-by-opportunity-stage-3031


# CallForge - 01 - Filter Gong Calls Synced to Salesforce by Opportunity Stage

### 1. Workflow Overview

This workflow, titled **CallForge - 01 - Filter Gong Calls Synced to Salesforce by Opportunity Stage**, is designed for sales and revenue teams using Gong and Salesforce. Its primary purpose is to automate the extraction, filtering, and preprocessing of Gong call data synced to Salesforce, preparing it for AI-driven analysis.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Data Retrieval:** Periodically triggers the workflow and fetches Gong call records from Salesforce custom objects.
- **1.2 Data Sorting and Filtering:** Sorts calls by creation date and filters them based on opportunity stage and presence of primary opportunity data.
- **1.3 Gong Call Details Retrieval:** Retrieves detailed Gong call information via the Gong API for filtered calls.
- **1.4 Data Formatting:** Structures the Gong call data into a JSON object suitable for AI processing.
- **1.5 Passing Data to AI Preprocessor:** Sends the formatted data to a dedicated Gong Call Preprocessor workflow for further analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Data Retrieval

- **Overview:**  
  This block initiates the workflow on a schedule (hourly) or manually, then queries Salesforce for Gong call custom objects created within the last 4 hours.

- **Nodes Involved:**  
  - Run Hourly  
  - When clicking ‘Test workflow’  
  - Get all custom Salesforce Gong Objects

- **Node Details:**

  - **Run Hourly**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow every hour.  
    - Configuration: Interval set to 1 hour.  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Get all custom Salesforce Gong Objects"  
    - Edge Cases: If the schedule is changed improperly, calls may be missed or duplicated.

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual triggering for testing purposes.  
    - Configuration: Default manual trigger with no parameters.  
    - Inputs: None  
    - Outputs: Connects to "Get all custom Salesforce Gong Objects"  
    - Edge Cases: Manual trigger may cause duplicate processing if used alongside schedule.

  - **Get all custom Salesforce Gong Objects**  
    - Type: Salesforce (customObject resource)  
    - Role: Retrieves Gong call records from Salesforce custom object `Gong__Gong_Call__c`.  
    - Configuration:  
      - Fields retrieved include CreatedDate, Gong Call ID, Opportunity info, and call metadata.  
      - Condition filters records with CreatedDate within the last 4 hours (`{{$now.minus(4, 'hours')}}`).  
    - Inputs: Trigger nodes  
    - Outputs: Connects to "Sort by date"  
    - Credentials: Salesforce OAuth2 credentials required.  
    - Edge Cases:  
      - API rate limits or auth failures.  
      - Query returning no records if no recent calls synced.  
      - Field names must match Salesforce schema exactly.

---

#### 2.2 Data Sorting and Filtering

- **Overview:**  
  This block sorts the retrieved calls by creation date descending and filters calls based on opportunity stage and presence of primary opportunity data.

- **Nodes Involved:**  
  - Sort by date  
  - Check if Opportunity Stage is Meeting Booked or Discovery  
  - Check if Primary Opportunity Contains Value

- **Node Details:**

  - **Sort by date**  
    - Type: Sort  
    - Role: Orders calls by `CreatedDate` descending to prioritize recent calls.  
    - Configuration: Sort field is `CreatedDate`, order descending.  
    - Inputs: "Get all custom Salesforce Gong Objects"  
    - Outputs: "Check if Opportunity Stage is Meeting Booked or Discovery"  
    - Edge Cases: Empty input array results in no further processing.

  - **Check if Opportunity Stage is Meeting Booked or Discovery**  
    - Type: If  
    - Role: Filters calls where `Gong__Opp_Stage_Time_Of_Call__c` equals "Discovery" or "Meeting Booked".  
    - Configuration: Logical OR condition on opportunity stage field.  
    - Inputs: "Sort by date"  
    - Outputs: True branch to "Check if Primary Opportunity Contains Value"  
    - Edge Cases:  
      - Field missing or null leads to false branch (calls filtered out).  
      - Case sensitivity enforced (must match exactly).

  - **Check if Primary Opportunity Contains Value**  
    - Type: If  
    - Role: Ensures the call has a non-empty `Gong__Primary_Opportunity__c` field.  
    - Configuration: Checks that the field is not empty.  
    - Inputs: True branch from previous node  
    - Outputs: True branch to "Get Gong Call"  
    - Edge Cases: Calls without primary opportunity are excluded.

---

#### 2.3 Gong Call Details Retrieval

- **Overview:**  
  For calls passing filters, this block retrieves detailed call data from Gong via API.

- **Nodes Involved:**  
  - Get Gong Call

- **Node Details:**

  - **Get Gong Call**  
    - Type: Gong API node  
    - Role: Fetches detailed Gong call information using Gong Call ID from Salesforce record.  
    - Configuration:  
      - Operation: Get call by ID  
      - Call ID sourced from `Gong__Call_ID__c` field of Salesforce record.  
    - Inputs: True branch from "Check if Primary Opportunity Contains Value"  
    - Outputs: "Format call into correct JSON Object"  
    - Credentials: Gong API credentials required.  
    - Edge Cases:  
      - API failures or invalid call IDs.  
      - Missing or malformed call data.

---

#### 2.4 Data Formatting

- **Overview:**  
  This block transforms the Gong call details into a structured JSON object with selected metadata fields, preparing it for AI processing.

- **Nodes Involved:**  
  - Format call into correct JSON Object

- **Node Details:**

  - **Format call into correct JSON Object**  
    - Type: Set  
    - Role: Maps Gong call metadata fields into a clean JSON structure.  
    - Configuration:  
      - Assigns multiple fields such as id, url, title, scheduled time, duration, primary user ID, call direction, language, workspace ID, and Salesforce opportunity ID (`sfOpp`).  
      - Uses expressions to extract data from the Gong call API response and Salesforce record.  
    - Inputs: "Get Gong Call"  
    - Outputs: "Pass to Gong Call Preprocessor"  
    - Edge Cases:  
      - Missing metadata fields may result in incomplete JSON.  
      - Expression errors if source data is malformed.

---

#### 2.5 Passing Data to AI Preprocessor

- **Overview:**  
  The final block sends the formatted call data to a separate workflow dedicated to AI preprocessing and analysis.

- **Nodes Involved:**  
  - Pass to Gong Call Preprocessor

- **Node Details:**

  - **Pass to Gong Call Preprocessor**  
    - Type: Execute Workflow  
    - Role: Invokes the "Gong Call Preprocessor Demo" workflow, passing the formatted call JSON for further AI-driven processing.  
    - Configuration:  
      - Workflow ID set to the preprocessor workflow.  
      - No additional parameters configured here.  
    - Inputs: "Format call into correct JSON Object"  
    - Outputs: None (end of this workflow)  
    - Edge Cases:  
      - Target workflow must exist and be accessible.  
      - Data format must match expected input schema of the preprocessor workflow.  
      - Failures in called workflow do not propagate here.

---

### 3. Summary Table

| Node Name                             | Node Type             | Functional Role                                  | Input Node(s)                      | Output Node(s)                             | Sticky Note                                                                                                                           |
|-------------------------------------|-----------------------|-------------------------------------------------|----------------------------------|--------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Run Hourly                          | Schedule Trigger      | Triggers workflow every hour                     | None                             | Get all custom Salesforce Gong Objects     | ![Callforge](https://uploads.n8n.io/templates/callforgeshadow.png) CallForge allows you to extract important information for different departments from your Sales Gong Calls. This workflow triggers the AI agent to run, processing calls every hour. It uses the Gong/Salesforce integration to look for new conversation objects in Salesforce which indicate that a new recording has synced to Salesforce. This allows us to filter calls based on internal milestones and metrics ensuring only calls that meet a certain criteria are processed. |
| When clicking ‘Test workflow’       | Manual Trigger        | Manual trigger for testing                       | None                             | Get all custom Salesforce Gong Objects     | Same as above                                                                                                                        |
| Get all custom Salesforce Gong Objects | Salesforce            | Retrieves Gong call records from Salesforce     | Run Hourly, When clicking ‘Test workflow’ | Sort by date                               |                                                                                                                                      |
| Sort by date                       | Sort                  | Sorts calls by creation date descending          | Get all custom Salesforce Gong Objects | Check if Opportunity Stage is Meeting Booked or Discovery | ## Get Gong Transcript and Call Details The transcript is to pass into the AI prompt, but needs to be transformed first. The Call details provide the Prompt with metadata. |
| Check if Opportunity Stage is Meeting Booked or Discovery | If                    | Filters calls by opportunity stage               | Sort by date                    | Check if Primary Opportunity Contains Value | Same as above                                                                                                                        |
| Check if Primary Opportunity Contains Value | If                    | Checks for non-empty primary opportunity field  | Check if Opportunity Stage is Meeting Booked or Discovery | Get Gong Call                               |                                                                                                                                      |
| Get Gong Call                      | Gong API               | Retrieves detailed Gong call data                 | Check if Primary Opportunity Contains Value | Format call into correct JSON Object       |                                                                                                                                      |
| Format call into correct JSON Object | Set                   | Formats Gong call data into structured JSON      | Get Gong Call                   | Pass to Gong Call Preprocessor              |                                                                                                                                      |
| Pass to Gong Call Preprocessor      | Execute Workflow      | Passes formatted data to AI preprocessing workflow | Format call into correct JSON Object | None                                       |                                                                                                                                      |
| Sticky Note5                       | Sticky Note            | Branding and workflow explanation                 | None                             | None                                       | See content in Run Hourly and When clicking ‘Test workflow’ rows                                                                  |
| Sticky Note4                       | Sticky Note            | Explanation of transcript and call details block | None                             | None                                       | See content in Sort by date and Check if Opportunity Stage is Meeting Booked or Discovery rows                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** node named `Run Hourly`.
     - Set interval to every 1 hour.
   - Add a **Manual Trigger** node named `When clicking ‘Test workflow’`.

2. **Add Salesforce Query Node:**
   - Add a **Salesforce** node named `Get all custom Salesforce Gong Objects`.
   - Set resource to `customObject` and operation to `getAll`.
   - Set custom object to `Gong__Gong_Call__c`.
   - Select fields: `CreatedDate`, `LastActivityDate`, `Name`, `Gong__Call_ID__c`, `Gong__Talk_Time_Us__c`, `Gong__Talk_Time_Them__c`, `Gong__Title__c`, `Gong__View_call__c`, `Gong__Primary_Opportunity__c`, `Gong__Opp_Stage_Time_Of_Call__c`.
   - Add condition: `CreatedDate >= {{$now.minus(4, 'hours')}}`.
   - Configure Salesforce OAuth2 credentials.

3. **Connect Trigger Nodes to Salesforce Node:**
   - Connect both `Run Hourly` and `When clicking ‘Test workflow’` nodes to `Get all custom Salesforce Gong Objects`.

4. **Add Sort Node:**
   - Add a **Sort** node named `Sort by date`.
   - Configure to sort by `CreatedDate` descending.
   - Connect `Get all custom Salesforce Gong Objects` output to this node.

5. **Add Opportunity Stage Filter:**
   - Add an **If** node named `Check if Opportunity Stage is Meeting Booked or Discovery`.
   - Configure condition (OR):  
     - `Gong__Opp_Stage_Time_Of_Call__c` equals "Discovery"  
     - OR `Gong__Opp_Stage_Time_Of_Call__c` equals "Meeting Booked"
   - Connect `Sort by date` output to this node.

6. **Add Primary Opportunity Check:**
   - Add an **If** node named `Check if Primary Opportunity Contains Value`.
   - Configure condition:  
     - `Gong__Primary_Opportunity__c` is not empty.
   - Connect the true output of `Check if Opportunity Stage is Meeting Booked or Discovery` to this node.

7. **Add Gong API Call Node:**
   - Add a **Gong** node named `Get Gong Call`.
   - Set operation to `get`.
   - Set call ID to `={{ $json.Gong__Call_ID__c }}` (expression referencing Salesforce record).
   - Configure Gong API credentials.
   - Connect true output of `Check if Primary Opportunity Contains Value` to this node.

8. **Add Data Formatting Node:**
   - Add a **Set** node named `Format call into correct JSON Object`.
   - Map the following fields from Gong call metadata and Salesforce record:  
     - `id`, `url`, `title`, `scheduled`, `started`, `duration`, `primaryUserId`, `direction`, `system`, `scope`, `media`, `language`, `workspaceId`, `sdrDisposition`, `clientUniqueId`, `customData`, `purpose`, `meetingUrl`, `isPrivate`, `calendarEventId`, `sfOpp` (from Salesforce `Gong__Primary_Opportunity__c`).
   - Use expressions to extract these values from `$json.metaData` and Salesforce fields.
   - Connect `Get Gong Call` output to this node.

9. **Add Execute Workflow Node:**
   - Add an **Execute Workflow** node named `Pass to Gong Call Preprocessor`.
   - Set workflow ID to the Gong Call Preprocessor workflow (e.g., "6mL5jWOJfuzkpjzx").
   - Connect `Format call into correct JSON Object` output to this node.

10. **Add Sticky Notes (Optional):**
    - Add sticky notes for branding and explanations as per original workflow.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| ![Callforge](https://uploads.n8n.io/templates/callforgeshadow.png) CallForge allows you to extract important information for different departments from your Sales Gong Calls. This workflow triggers the AI agent to run, processing calls every hour. It uses the Gong/Salesforce integration to look for new conversation objects in Salesforce which indicate that a new recording has synced to Salesforce. This allows us to filter calls based on internal milestones and metrics ensuring only calls that meet a certain criteria are processed. | Branding and workflow explanation sticky note content                                           |
| ## Get Gong Transcript and Call Details The transcript is to pass into the AI prompt, but needs to be transformed first. The Call details provide the Prompt with metadata.                                                                                                                                                                                                              | Explanation sticky note content covering the data retrieval and formatting block                |
| Ensure Salesforce and Gong API credentials are valid and have appropriate permissions for reading call data and custom objects.                                                                                                                                                                                                                                                         | Credential setup note                                                                          |
| Modify Salesforce query and filtering conditions to match your organization's sales process and Gong integration schema.                                                                                                                                                                                                                                                                 | Customization advice                                                                           |
| Connect this workflow’s output to an AI processing workflow to analyze call transcripts and generate insights.                                                                                                                                                                                                                                                                             | Integration advice                                                                            |

---

This documentation provides a comprehensive understanding of the CallForge workflow, enabling users and AI agents to reproduce, modify, and troubleshoot it effectively.