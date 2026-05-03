CallForge - 04 - AI Workflow for Gong.io Sales Calls

https://n8nworkflows.xyz/workflows/callforge---04---ai-workflow-for-gong-io-sales-calls-3034


# CallForge - 04 - AI Workflow for Gong.io Sales Calls

### 1. Workflow Overview

**CallForge - AI Gong Sales Call Processing Workflow** automates the ingestion, processing, and AI-driven analysis of Gong.io sales call data. It targets sales teams, RevOps professionals, and AI sales intelligence teams who want to scale call data processing, integrate insights into Notion, and receive real-time Slack notifications.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Initial Filtering**: Trigger on new Gong calls, retrieve existing Notion records, and filter out already processed calls to avoid duplicates.
- **1.2 Parent Notion Record Creation**: For each new call, create a parent record in Notion containing call metadata and Salesforce opportunity details.
- **1.3 Call Processing Loop**: Process calls one-by-one to handle API rate limits and allow resuming from failures.
- **1.4 AI-Powered Call Analysis**: Pass call data and Notion parent IDs into a subworkflow that performs AI-driven analysis and data extraction.
- **1.5 Real-Time Slack Notifications**: Send Slack alerts on queue start, progress updates during processing, and completion notifications.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initial Filtering

- **Overview:**  
  This block triggers the workflow on new Gong calls, retrieves all previously processed calls from Notion, and filters out calls already stored to prevent duplication.

- **Nodes Involved:**  
  - Execute Workflow Trigger  
  - Reduce down to 1 object (aggregate)  
  - Get all older Calls (Notion)  
  - Isolate Only Call IDs (Set)  
  - Only Process New Calls (Compare Datasets)  
  - Reduce down to One object (aggregate)  
  - Post Slack Receipt (Slack)  
  - Bundle Slack Message Data (aggregate)  
  - Merge Slack and Call Data (merge)

- **Node Details:**

  - **Execute Workflow Trigger**  
    - *Type:* Execute Workflow Trigger  
    - *Role:* Entry point that starts the workflow when new Gong call data arrives.  
    - *Config:* Default trigger, no parameters.  
    - *Inputs:* External trigger event (new Gong call).  
    - *Outputs:* Passes new call data downstream.  
    - *Edge Cases:* Trigger may fail if Gong API credentials are invalid or if no new calls arrive.

  - **Reduce down to 1 object (aggregate)**  
    - *Type:* Aggregate  
    - *Role:* Aggregates incoming data items into a single object for easier processing.  
    - *Config:* Aggregates all item data into one object.  
    - *Inputs:* New Gong call data.  
    - *Outputs:* Single aggregated object.  
    - *Edge Cases:* Empty input array results in no output.

  - **Get all older Calls (Notion)**  
    - *Type:* Notion (databasePage, getAll)  
    - *Role:* Retrieves all existing call records from the Notion database to check for duplicates.  
    - *Config:* Uses Notion database ID for "Sales Call Summaries Demo".  
    - *Inputs:* Triggered by previous aggregate node.  
    - *Outputs:* List of all call pages in Notion.  
    - *Edge Cases:* Notion API rate limits or auth failures; large datasets may cause timeouts.

  - **Isolate Only Call IDs (Set)**  
    - *Type:* Set  
    - *Role:* Extracts only the Gong Call IDs from Notion records for comparison.  
    - *Config:* Sets a field "Call ID" from property_gong_call_id or defaults to "none".  
    - *Inputs:* Notion call records.  
    - *Outputs:* List of call IDs.  
    - *Edge Cases:* Missing or malformed Gong Call IDs.

  - **Only Process New Calls (Compare Datasets)**  
    - *Type:* Compare Datasets  
    - *Role:* Compares new Gong calls against existing Notion call IDs to filter only new calls.  
    - *Config:* Merges by GongCallID (input1) and Call ID (input2), preferring input1 data.  
    - *Inputs:* New calls and existing call IDs.  
    - *Outputs:* Only new calls not in Notion.  
    - *Edge Cases:* Mismatched or missing IDs may cause false positives/negatives.

  - **Reduce down to One object (aggregate)**  
    - *Type:* Aggregate  
    - *Role:* Aggregates filtered new calls into a single object.  
    - *Config:* Aggregate all item data.  
    - *Inputs:* Filtered new calls.  
    - *Outputs:* Single aggregated object of new calls.  
    - *Edge Cases:* Empty input means no new calls to process.

  - **Post Slack Receipt (Slack)**  
    - *Type:* Slack (post message)  
    - *Role:* Sends a Slack message indicating the queue has started processing new calls.  
    - *Config:* Posts to channel "project-call-forge-alerts" with message including number of calls.  
    - *Inputs:* Aggregated new calls.  
    - *Outputs:* Slack message metadata.  
    - *Edge Cases:* Slack API auth failure or channel ID invalid.

  - **Bundle Slack Message Data (aggregate)**  
    - *Type:* Aggregate  
    - *Role:* Aggregates Slack message metadata for later use in progress updates.  
    - *Config:* Aggregates all item data into "slackdata" field.  
    - *Inputs:* Slack message metadata.  
    - *Outputs:* Aggregated Slack data.  
    - *Edge Cases:* Slack message failure results in missing data.

  - **Merge Slack and Call Data (merge)**  
    - *Type:* Merge  
    - *Role:* Combines Slack message data and new call data into one dataset for downstream processing.  
    - *Config:* Combine all inputs.  
    - *Inputs:* Aggregated Slack data and new calls.  
    - *Outputs:* Combined dataset.  
    - *Edge Cases:* Mismatched data lengths or missing inputs.

---

#### 2.2 Parent Notion Record Creation

- **Overview:**  
  For each new call, this block creates a parent record in Notion with detailed metadata and Salesforce opportunity information. This parent record serves as the anchor for AI processing and related data objects.

- **Nodes Involved:**  
  - Loop Over Calls (splitInBatches)  
  - Create Notion DB Page (Notion)  
  - Bundle Notion Parent Object Data (aggregate)  
  - Merge call data and parent notion id (merge)

- **Node Details:**

  - **Loop Over Calls (splitInBatches)**  
    - *Type:* Split In Batches  
    - *Role:* Processes calls one at a time to avoid Notion API rate limits.  
    - *Config:* Default batch size = 1.  
    - *Inputs:* Combined Slack and call data.  
    - *Outputs:* Single call per batch.  
    - *Edge Cases:* Large call volumes may slow processing.

  - **Create Notion DB Page (Notion)**  
    - *Type:* Notion (databasePage, create)  
    - *Role:* Creates a new page in Notion for each call with detailed properties.  
    - *Config:*  
      - Title: call title from metadata  
      - Icon: phone emoji 📞  
      - Properties: Call Date, Recording URL, Company, Call Name, Gong Call ID, Salesforce Opportunity ID, Stage, Company ID, Won/Closed flags, Company Size, Sales Rep(s), Salesforce links  
      - Uses Notion API credentials "Angelbot Notion"  
      - Retry on failure with 3s delay (to handle rate limits)  
    - *Inputs:* Single call data from loop.  
    - *Outputs:* Newly created Notion page data.  
    - *Edge Cases:* Notion API rate limiting, invalid property values, missing Salesforce data.

  - **Bundle Notion Parent Object Data (aggregate)**  
    - *Type:* Aggregate  
    - *Role:* Aggregates all created Notion parent pages into one collection.  
    - *Config:* Aggregate all item data into "notionData" field.  
    - *Inputs:* Created Notion pages.  
    - *Outputs:* Aggregated Notion parent data.  
    - *Edge Cases:* Empty input if no calls processed.

  - **Merge call data and parent notion id (merge)**  
    - *Type:* Merge  
    - *Role:* Combines original call data with the corresponding Notion parent page data for downstream AI processing.  
    - *Config:* Combine by position (one-to-one).  
    - *Inputs:* Aggregated Notion data and original call data.  
    - *Outputs:* Combined dataset with call and Notion parent info.  
    - *Edge Cases:* Mismatched array lengths could cause data misalignment.

---

#### 2.3 Call Processing Loop and AI Analysis

- **Overview:**  
  This block sends each call with its Notion parent ID into an AI-powered subworkflow for detailed analysis and structured data extraction. It also manages looping and progress updates.

- **Nodes Involved:**  
  - AI Team Processor (Execute Workflow)  
  - Update Slack Progress (Slack)  
  - Loop to next call (NoOp)  

- **Node Details:**

  - **AI Team Processor (Execute Workflow)**  
    - *Type:* Execute Workflow  
    - *Role:* Invokes a subworkflow ("AI Team Processor Demo") that performs AI-driven call analysis and data extraction.  
    - *Config:* Workflow ID selected from list.  
    - *Inputs:* Merged call and Notion parent data.  
    - *Outputs:* Processed AI results.  
    - *Edge Cases:* Subworkflow failures, AI API timeouts, invalid input data.

  - **Update Slack Progress (Slack)**  
    - *Type:* Slack (update message)  
    - *Role:* Updates the Slack message with current processing progress (e.g., "Processing call 3 of 10").  
    - *Config:*  
      - Uses timestamp from initial Slack message to update same thread  
      - Channel: "project-call-forge-alerts"  
      - Text includes current index and total calls  
    - *Inputs:* AI processor output and loop context.  
    - *Outputs:* Updated Slack message metadata.  
    - *Edge Cases:* Slack API rate limits, invalid timestamp, message update failures.

  - **Loop to next call (NoOp)**  
    - *Type:* No Operation (NoOp)  
    - *Role:* Acts as a connector to loop back to "Loop Over Calls" node to process the next batch.  
    - *Config:* None.  
    - *Inputs:* From Slack progress update.  
    - *Outputs:* Triggers next batch processing.  
    - *Edge Cases:* Infinite loops if not properly controlled.

---

#### 2.4 Completion Notification

- **Overview:**  
  After all calls are processed, this block aggregates processed calls and sends a final Slack notification indicating successful completion.

- **Nodes Involved:**  
  - Bundle Processed Calls (aggregate)  
  - Post Completed Calls Message (Slack)

- **Node Details:**

  - **Bundle Processed Calls (aggregate)**  
    - *Type:* Aggregate  
    - *Role:* Aggregates all processed call data into one collection for summary.  
    - *Config:* Aggregate all item data.  
    - *Inputs:* Output from "Loop Over Calls".  
    - *Outputs:* Aggregated processed calls.  
    - *Edge Cases:* Empty input if no calls processed.

  - **Post Completed Calls Message (Slack)**  
    - *Type:* Slack (post message)  
    - *Role:* Posts a Slack message indicating the queue has finished processing all calls successfully.  
    - *Config:*  
      - Channel: "project-call-forge-alerts"  
      - Message includes count of calls processed  
    - *Inputs:* Aggregated processed calls.  
    - *Outputs:* Slack message metadata.  
    - *Edge Cases:* Slack API failures, invalid channel.

---

### 3. Summary Table

| Node Name                     | Node Type                 | Functional Role                                  | Input Node(s)                       | Output Node(s)                      | Sticky Note                                                                                          |
|-------------------------------|---------------------------|-------------------------------------------------|-----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------|
| Execute Workflow Trigger       | Execute Workflow Trigger  | Entry trigger on new Gong calls                  | -                                 | Only Process New Calls, Reduce down to 1 object | ![Callforge](https://uploads.n8n.io/templates/callforgeshadow.png) CallForge intro and parent object creation explanation |
| Reduce down to 1 object        | Aggregate                 | Aggregate new Gong calls into one object         | Execute Workflow Trigger           | Get all older Calls                |                                                                                                    |
| Get all older Calls            | Notion                    | Retrieve all existing calls from Notion          | Reduce down to 1 object            | Isolate Only Call IDs              |                                                                                                    |
| Isolate Only Call IDs          | Set                       | Extract Gong Call IDs from Notion records        | Get all older Calls                | Only Process New Calls             |                                                                                                    |
| Only Process New Calls         | Compare Datasets          | Filter out calls already processed in Notion     | Execute Workflow Trigger, Isolate Only Call IDs | Reduce down to One object, Merge Slack and Call Data | Process queue logic: rerun on remaining calls to handle Notion rate limits                         |
| Reduce down to One object      | Aggregate                 | Aggregate filtered new calls                      | Only Process New Calls             | Post Slack Receipt                |                                                                                                    |
| Post Slack Receipt             | Slack                     | Notify Slack queue started                        | Reduce down to One object          | Bundle Slack Message Data          |                                                                                                    |
| Bundle Slack Message Data      | Aggregate                 | Aggregate Slack message metadata                  | Post Slack Receipt                | Merge Slack and Call Data          |                                                                                                    |
| Merge Slack and Call Data      | Merge                     | Combine Slack data and new calls                  | Bundle Slack Message Data, Only Process New Calls | Loop Over Calls                   | Loop over calls for analysis and create parent DB object for AI processing                         |
| Loop Over Calls               | Split In Batches          | Process calls one at a time                        | Merge Slack and Call Data          | Bundle Processed Calls, Merge call data and parent notion id, Create Notion DB Page |                                                                                                    |
| Create Notion DB Page          | Notion                    | Create parent Notion page for each call           | Loop Over Calls                   | Bundle Notion Parent Object Data   |                                                                                                    |
| Bundle Notion Parent Object Data | Aggregate               | Aggregate created Notion parent pages             | Create Notion DB Page             | Merge call data and parent notion id |                                                                                                    |
| Merge call data and parent notion id | Merge               | Combine call data with Notion parent page data    | Bundle Notion Parent Object Data, Loop Over Calls | AI Team Processor               | Pass parent Notion ID and call data into AI subworkflow for final prompt processing                |
| AI Team Processor             | Execute Workflow          | Invoke AI subworkflow for call analysis           | Merge call data and parent notion id | Update Slack Progress             |                                                                                                    |
| Update Slack Progress          | Slack                     | Update Slack message with processing progress     | AI Team Processor                 | Loop to next call                 | Alert on progress: real-time Slack updates                                                        |
| Loop to next call             | No Operation (NoOp)       | Loop control to process next call                  | Update Slack Progress             | Loop Over Calls                   |                                                                                                    |
| Bundle Processed Calls         | Aggregate                 | Aggregate all processed calls                       | Loop Over Calls                  | Post Completed Calls Message       |                                                                                                    |
| Post Completed Calls Message   | Slack                     | Notify Slack queue completion                       | Bundle Processed Calls            | -                                 | Alert Slack job complete: notify team of successful run                                           |
| Sticky Note                   | Sticky Note               | Various notes explaining blocks and logic          | -                                 | -                                 | Multiple sticky notes with explanations and branding (see detailed notes in section 5)            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node**  
   - Add an **Execute Workflow Trigger** node as the entry point to receive new Gong call data.

2. **Aggregate Incoming Calls**  
   - Add an **Aggregate** node ("Reduce down to 1 object") to combine incoming call data into a single object.

3. **Retrieve Existing Calls from Notion**  
   - Add a **Notion** node ("Get all older Calls") configured to get all pages from your Gong calls Notion database.  
   - Use your Notion API credentials and specify the database ID.

4. **Extract Gong Call IDs from Notion Data**  
   - Add a **Set** node ("Isolate Only Call IDs") to extract the Gong Call ID property from each Notion page.  
   - Configure to set a field "Call ID" from `property_gong_call_id` or default to "none".

5. **Filter New Calls Only**  
   - Add a **Compare Datasets** node ("Only Process New Calls") to compare new Gong calls with existing Notion call IDs.  
   - Configure to merge by `metaData.GongCallID` (input1) and `Call ID` (input2), preferring input1 data.

6. **Aggregate Filtered Calls**  
   - Add an **Aggregate** node ("Reduce down to One object") to aggregate only new calls into one object.

7. **Send Slack Notification - Queue Started**  
   - Add a **Slack** node ("Post Slack Receipt") to post a message to your Slack channel indicating the number of calls to process.  
   - Configure with Slack API credentials and target channel.

8. **Aggregate Slack Message Data**  
   - Add an **Aggregate** node ("Bundle Slack Message Data") to collect Slack message metadata for progress updates.

9. **Merge Slack and Call Data**  
   - Add a **Merge** node ("Merge Slack and Call Data") to combine Slack message data and new call data.

10. **Split Calls for Sequential Processing**  
    - Add a **Split In Batches** node ("Loop Over Calls") with batch size 1 to process calls one at a time.

11. **Create Parent Notion Record for Each Call**  
    - Add a **Notion** node ("Create Notion DB Page") configured to create a new page in your Notion database.  
    - Map call metadata and Salesforce opportunity details to Notion properties.  
    - Use retry on fail with a 3-second delay to handle rate limits.

12. **Aggregate Created Notion Pages**  
    - Add an **Aggregate** node ("Bundle Notion Parent Object Data") to collect all created Notion pages.

13. **Merge Call Data with Notion Parent IDs**  
    - Add a **Merge** node ("Merge call data and parent notion id") to combine original call data with created Notion page data by position.

14. **Invoke AI Processing Subworkflow**  
    - Add an **Execute Workflow** node ("AI Team Processor") to call your AI analysis subworkflow.  
    - Pass merged call and Notion parent data as input.

15. **Update Slack Progress**  
    - Add a **Slack** node ("Update Slack Progress") to update the Slack message with current processing progress.  
    - Use the timestamp from the initial Slack message to update the same message thread.

16. **Loop Control**  
    - Add a **No Operation (NoOp)** node ("Loop to next call") connected from Slack progress update.  
    - Connect this node back to "Loop Over Calls" to process the next call batch.

17. **Aggregate Processed Calls**  
    - Add an **Aggregate** node ("Bundle Processed Calls") to collect all processed call results.

18. **Send Slack Completion Notification**  
    - Add a **Slack** node ("Post Completed Calls Message") to notify the Slack channel that processing is complete.

19. **Connect All Nodes**  
    - Ensure all nodes are connected as per the logical flow described above.

20. **Configure Credentials**  
    - Set up and assign credentials for:  
      - Gong API (used externally to trigger workflow)  
      - Notion API (for database access)  
      - Slack API (for posting messages)  
      - AI subworkflow credentials (e.g., OpenAI or custom AI service)

21. **Test Workflow**  
    - Run the workflow with test Gong call data to verify each step completes successfully and Slack notifications appear as expected.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                      |
|----------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| ![Callforge](https://uploads.n8n.io/templates/callforgeshadow.png) CallForge branding and intro image.  | Workflow branding image used in sticky notes.                                                      |
| Process queue logic: rerun on remaining calls to handle Notion API rate limits gracefully.                | Sticky Note explaining resilience and rerun capability.                                            |
| Loop over calls for analysis and create parent DB object to relate other DB objects to.                   | Sticky Note describing the creation of parent Notion records for AI processing.                     |
| Pass parent Notion ID and call data into AI subworkflow for final prompt processing.                      | Sticky Note describing AI subworkflow invocation for structured data extraction.                    |
| Alert on progress: real-time Slack updates keep the team informed on processing status.                   | Sticky Note describing Slack progress notifications.                                               |
| Alert Slack job complete: notify team of successful workflow run completion.                              | Sticky Note describing final Slack notification on job completion.                                 |
| CallForge allows extraction of important information for different departments from Gong sales calls.    | Workflow description emphasizing multi-departmental AI insights extraction.                        |
| Modify Notion integration to use other CRMs like HubSpot, Airtable, or Salesforce as needed.              | Customization tip for adapting the workflow to other CRM platforms.                                |
| Customize Slack notifications or replace with email alerts as preferred.                                 | Customization tip for notification channels.                                                       |
| Expand workflow with integrations to Salesforce, Pipedrive, or HubSpot for enriched sales data.          | Suggestion for extending workflow capabilities with additional sales tools.                        |

---

This documentation provides a complete, structured reference for understanding, reproducing, and customizing the CallForge AI Gong Sales Call Processing workflow in n8n.