CallForge - 07 - AI Marketing Data Processing with Gong & Notion

https://n8nworkflows.xyz/workflows/callforge---07---ai-marketing-data-processing-with-gong---notion-3037


# CallForge - 07 - AI Marketing Data Processing with Gong & Notion

### 1. Workflow Overview

This workflow, **CallForge - AI Marketing Data Processing with Gong & Notion**, automates the extraction, categorization, and storage of marketing insights from AI-analyzed sales call data. It is designed to streamline marketing intelligence gathering by processing AI-generated sales call outputs, identifying key marketing trends, recurring topics, and actionable recommendations, and then logging these insights into structured Notion databases.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives AI-processed sales call data via a workflow trigger.
- **1.2 Marketing Insights Processing:** Detects and processes marketing-related insights from the AI data, storing them in the "Marketing Insights" Notion database.
- **1.3 Recurring Topics Processing:** Identifies recurring discussion topics across calls, processes them, and stores them in the "Recurring Topics" Notion database.
- **1.4 Actionable Insights Processing:** Extracts actionable marketing recommendations and logs them into the "Actionable Recommendations" Notion database.
- **1.5 Data Aggregation and Thread Merging:** Aggregates processed data and merges AI response threads for each category to maintain context and continuity.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by receiving AI-generated sales call data from an external source or another workflow. It acts as the entry point for all subsequent processing.

- **Nodes Involved:**  
  - Execute Workflow Trigger  
  - Sticky Note6 (comment)  
  - Sticky Note7 (branding and overview)

- **Node Details:**  
  - **Execute Workflow Trigger**  
    - Type: `Execute Workflow Trigger`  
    - Role: Entry point node that triggers the workflow when AI sales call data is received.  
    - Configuration: No parameters; expects incoming JSON with AI call analysis data and metadata.  
    - Inputs: External webhook or upstream workflow trigger.  
    - Outputs: Passes the AI data downstream for processing.  
    - Edge Cases: Trigger failure if data is malformed or missing expected fields; ensure upstream workflow sends correct data format.

- **Sticky Notes:**  
  - Sticky Note6: "Receives AI Data from other workflow" — clarifies the trigger’s role.  
  - Sticky Note7: Branding and high-level description of CallForge’s purpose and AI output processing.

---

#### 2.2 Marketing Insights Processing

- **Overview:**  
  This block detects if marketing insights exist in the AI data, processes each insight individually, creates corresponding entries in the "Marketing Insights" Notion database, and aggregates the results.

- **Nodes Involved:**  
  - Check if Marketing Insight Data Found (If node)  
  - Wait for rate limiting - Marketing Insights (Wait node)  
  - Split out Insights (SplitOut node)  
  - Create Marketing Insight Data (Notion node)  
  - Bundle Marketing Insights Data to 1 object (Aggregate node)  
  - Merge Marketing Insights Thread (Set node)  
  - Sticky Note5 (comment)

- **Node Details:**  
  - **Check if Marketing Insight Data Found**  
    - Type: `If`  
    - Role: Checks if the AI data contains an array with at least one marketing insight.  
    - Condition: Length of `AIoutput.MarketingInsights` array ≥ 1.  
    - Outputs: True branch proceeds with processing; false branch skips.  
    - Edge Cases: Empty or missing `MarketingInsights` field leads to skipping this block.

  - **Wait for rate limiting - Marketing Insights**  
    - Type: `Wait`  
    - Role: Adds a 3-second delay to avoid API rate limits when writing to Notion.  
    - Configuration: Wait 3 seconds before continuing.  
    - Edge Cases: Delay may impact throughput but prevents API throttling.

  - **Split out Insights**  
    - Type: `SplitOut`  
    - Role: Splits the array of marketing insights into individual items for separate processing.  
    - Field to split: `AIoutput.MarketingInsights`  
    - Edge Cases: Empty array leads to no output; ensure input is valid array.

  - **Create Marketing Insight Data**  
    - Type: `Notion` (databasePage creation)  
    - Role: Creates a new page in the "Marketing Insights" Notion database for each insight.  
    - Key Configurations:  
      - Title: Uses the call title from trigger metadata.  
      - Properties mapped:  
        - Name (title): `Summary` field from insight JSON  
        - Marketing Tags (multi-select): `Tag` field from insight JSON  
        - Sales Call Summaries (relation): Links to the original sales call summary in Notion (using ID from trigger data)  
        - Date Mentioned (date): Call start time from trigger metadata  
      - Icon: 🎯  
    - Credentials: Uses "Notion david-internal" API credentials.  
    - Retry: Enabled with 3-second wait between tries.  
    - Edge Cases: API errors, missing fields, or invalid Notion database schema can cause failures.

  - **Bundle Marketing Insights Data to 1 object**  
    - Type: `Aggregate`  
    - Role: Aggregates all created marketing insight items back into a single object for further processing or merging.  
    - Destination field: `tagdata`  
    - Edge Cases: No items to aggregate if previous steps failed.

  - **Merge Marketing Insights Thread**  
    - Type: `Set`  
    - Role: Adds the original AI response object from the trigger to the aggregated data for context.  
    - Assigns: `aiResponse` = AI response JSON from trigger.  
    - Edge Cases: Missing AI response data would reduce context.

  - **Sticky Note5:** "Actionable Insights" (positioned near this block, but more relevant to actionable insights block).

---

#### 2.3 Recurring Topics Processing

- **Overview:**  
  This block identifies recurring topics mentioned across multiple sales calls, processes each topic, creates entries in the "Recurring Topics" Notion database, and aggregates the results.

- **Nodes Involved:**  
  - Check if Recurring Topics Found (If node)  
  - Wait for rate limiting - Recurring (Wait node)  
  - Split Out Recurring Topics (SplitOut node)  
  - Create Recurring Topics Data (Notion node)  
  - Bundle Recurring Topics Data to 1 object (Aggregate node)  
  - Merge Recurring Topics Thread (Set node)  
  - Sticky Note8 (comment)

- **Node Details:**  
  - **Check if Recurring Topics Found**  
    - Type: `If`  
    - Role: Checks if the AI data contains an array with at least one recurring topic.  
    - Condition: Length of `AIoutput.MarketingInsights` array ≥ 1 (note: condition uses MarketingInsights field, likely a minor misconfiguration; expected to check `AIoutput.RecurringTopics`).  
    - Edge Cases: If condition misconfigured, recurring topics may be skipped.

  - **Wait for rate limiting - Recurring**  
    - Type: `Wait`  
    - Role: 3-second delay to avoid Notion API rate limits.  
    - Edge Cases: Same as previous wait nodes.

  - **Split Out Recurring Topics**  
    - Type: `SplitOut`  
    - Role: Splits the `AIoutput.RecurringTopics` array into individual topic items.  
    - Edge Cases: Empty or missing array leads to no output.

  - **Create Recurring Topics Data**  
    - Type: `Notion` (databasePage creation)  
    - Role: Creates a new page in the "Recurring Topics" Notion database for each topic.  
    - Key Configurations:  
      - Title: `Topic` field from JSON  
      - Properties mapped:  
        - Context (rich text): `Context` field  
        - Mentions (number): `Mentions` field  
        - Sales Call Summaries (relation): Links to original call summary ID from trigger  
      - Icon: 🔁  
    - Credentials: Uses "Angelbot Notion" API credentials.  
    - Retry: Enabled with 3-second wait between tries.  
    - Edge Cases: API errors, missing fields, or schema mismatches.

  - **Bundle Recurring Topics Data to 1 object**  
    - Type: `Aggregate`  
    - Role: Aggregates all created recurring topic items into a single object.  
    - Destination field: `tagdata`  
    - Edge Cases: No items to aggregate if previous steps failed.

  - **Merge Recurring Topics Thread**  
    - Type: `Set`  
    - Role: Adds the original AI response object from the trigger to the aggregated data for context.  
    - Assigns: `aiResponse` = AI response JSON from trigger.  
    - Edge Cases: Missing AI response data reduces context.

  - **Sticky Note8:** "Recurring Topics" — labels this block.

---

#### 2.4 Actionable Insights Processing

- **Overview:**  
  This block processes actionable marketing recommendations extracted from AI data, creates entries in the "Actionable Recommendations" Notion database, and aggregates the results.

- **Nodes Involved:**  
  - Check if Actionable Insights Data Found (If node)  
  - Wait for rate limiting - Actionable Insights (Wait node)  
  - Split Out Actionable Insights (SplitOut node)  
  - Create Actionable Insights Data (Notion node)  
  - Bundle Actionable Insights Data to 1 object (Aggregate node)  
  - Merge Actionable Insights Thread (Set node)  
  - Sticky Note (comment near this block)

- **Node Details:**  
  - **Check if Actionable Insights Data Found**  
    - Type: `If`  
    - Role: Checks if the AI data contains an array with at least one actionable insight.  
    - Condition: Length of `AIoutput.ActionableInsights` array ≥ 1.  
    - Edge Cases: Empty or missing actionable insights skips this block.

  - **Wait for rate limiting - Actionable Insights**  
    - Type: `Wait`  
    - Role: 3-second delay to avoid Notion API rate limits.  
    - Edge Cases: Same as previous wait nodes.

  - **Split Out Actionable Insights**  
    - Type: `SplitOut`  
    - Role: Splits the `AIoutput.ActionableInsights` array into individual actionable insight items.  
    - Edge Cases: Empty or missing array leads to no output.

  - **Create Actionable Insights Data**  
    - Type: `Notion` (databasePage creation)  
    - Role: Creates a new page in the "Actionable Recommendations" Notion database for each actionable insight.  
    - Key Configurations:  
      - Title: `Topic` field from JSON  
      - Properties mapped:  
        - Rationale (rich text): `Rationale` field  
        - Recommendation Type (rich text): `RecommendationType` field  
        - Title (rich text): `Title` field  
        - Sales Call Summaries (relation): Links to original call summary ID from trigger  
      - Icon: 🎬  
    - Credentials: Uses "Angelbot Notion" API credentials.  
    - Retry: Enabled with 3-second wait between tries.  
    - Edge Cases: API errors, missing fields, or schema mismatches.

  - **Bundle Actionable Insights Data to 1 object**  
    - Type: `Aggregate`  
    - Role: Aggregates all created actionable insight items into a single object.  
    - Destination field: `tagdata`  
    - Edge Cases: No items to aggregate if previous steps failed.

  - **Merge Actionable Insights Thread**  
    - Type: `Set`  
    - Role: Adds the original AI response object from the trigger to the aggregated data for context.  
    - Assigns: `aiResponse` = AI response JSON from trigger.  
    - Edge Cases: Missing AI response data reduces context.

  - **Sticky Note:** "Actionable Insights" — labels this block.

---

#### 2.5 Data Aggregation and Thread Merging (Cross-Cutting)

- **Overview:**  
  Each processing block aggregates its created Notion entries and merges the AI response thread context to maintain continuity and enable downstream usage or reporting.

- **Nodes Involved:**  
  - Bundle Marketing Insights Data to 1 object  
  - Merge Marketing Insights Thread  
  - Bundle Recurring Topics Data to 1 object  
  - Merge Recurring Topics Thread  
  - Bundle Actionable Insights Data to 1 object  
  - Merge Actionable Insights Thread

- **Node Details:**  
  - Aggregation nodes collect multiple created Notion pages into a single data object.  
  - Merge nodes append the original AI response JSON to the aggregated data for traceability.  
  - These nodes ensure that each category’s data is consolidated and contextually enriched.

---

### 3. Summary Table

| Node Name                          | Node Type                  | Functional Role                                  | Input Node(s)                   | Output Node(s)                          | Sticky Note                                                                                  |
|----------------------------------|----------------------------|-------------------------------------------------|--------------------------------|---------------------------------------|----------------------------------------------------------------------------------------------|
| Execute Workflow Trigger          | Execute Workflow Trigger   | Entry point receiving AI sales call data        | External trigger               | Check if Marketing Insight Data Found, Check if Recurring Topics Found, Check if Actionable Insights Data Found | Receives AI Data from other workflow (Sticky Note6)                                         |
| Check if Marketing Insight Data Found | If                        | Checks presence of marketing insights            | Execute Workflow Trigger       | Wait for rate limiting - Marketing Insights, Merge Marketing Insights Thread |                                                                                              |
| Wait for rate limiting - Marketing Insights | Wait                      | Rate limit delay before Notion API calls         | Check if Marketing Insight Data Found | Split out Insights                    |                                                                                              |
| Split out Insights               | SplitOut                   | Splits marketing insights array into items       | Wait for rate limiting - Marketing Insights | Create Marketing Insight Data         |                                                                                              |
| Create Marketing Insight Data    | Notion                     | Creates Notion pages for marketing insights      | Split out Insights             | Bundle Marketing Insights Data to 1 object | Marketing Insights Processing (Sticky Note5)                                                |
| Bundle Marketing Insights Data to 1 object | Aggregate                  | Aggregates created marketing insight pages       | Create Marketing Insight Data  | Merge Marketing Insights Thread       |                                                                                              |
| Merge Marketing Insights Thread  | Set                        | Adds AI response context to aggregated data      | Bundle Marketing Insights Data to 1 object | -                                     |                                                                                              |
| Check if Recurring Topics Found  | If                         | Checks presence of recurring topics               | Execute Workflow Trigger       | Wait for rate limiting - Recurring, Merge Recurring Topics Thread | Recurring Topics (Sticky Note8)                                                             |
| Wait for rate limiting - Recurring | Wait                      | Rate limit delay before Notion API calls         | Check if Recurring Topics Found | Split Out Recurring Topics            |                                                                                              |
| Split Out Recurring Topics       | SplitOut                   | Splits recurring topics array into items          | Wait for rate limiting - Recurring | Create Recurring Topics Data          |                                                                                              |
| Create Recurring Topics Data     | Notion                     | Creates Notion pages for recurring topics         | Split Out Recurring Topics     | Bundle Recurring Topics Data to 1 object |                                                                                              |
| Bundle Recurring Topics Data to 1 object | Aggregate                  | Aggregates created recurring topic pages          | Create Recurring Topics Data   | Merge Recurring Topics Thread          |                                                                                              |
| Merge Recurring Topics Thread    | Set                        | Adds AI response context to aggregated data      | Bundle Recurring Topics Data to 1 object | -                                     |                                                                                              |
| Check if Actionable Insights Data Found | If                        | Checks presence of actionable insights            | Execute Workflow Trigger       | Wait for rate limiting - Actionable Insights, Merge Actionable Insights Thread | Actionable Insights (Sticky Note)                                                           |
| Wait for rate limiting - Actionable Insights | Wait                      | Rate limit delay before Notion API calls         | Check if Actionable Insights Data Found | Split Out Actionable Insights         |                                                                                              |
| Split Out Actionable Insights    | SplitOut                   | Splits actionable insights array into items       | Wait for rate limiting - Actionable Insights | Create Actionable Insights Data       |                                                                                              |
| Create Actionable Insights Data  | Notion                     | Creates Notion pages for actionable insights      | Split Out Actionable Insights  | Bundle Actionable Insights Data to 1 object |                                                                                              |
| Bundle Actionable Insights Data to 1 object | Aggregate                  | Aggregates created actionable insight pages       | Create Actionable Insights Data | Merge Actionable Insights Thread       |                                                                                              |
| Merge Actionable Insights Thread | Set                        | Adds AI response context to aggregated data      | Bundle Actionable Insights Data to 1 object | -                                     |                                                                                              |
| Sticky Note5                    | Sticky Note                | Labels Marketing Insights Processing block        | -                              | -                                     | "Marketing Insights Processing"                                                             |
| Sticky Note6                    | Sticky Note                | Labels Input Reception block                        | -                              | -                                     | "Receives AI Data from other workflow"                                                      |
| Sticky Note7                    | Sticky Note                | Branding and overview of CallForge workflow        | -                              | -                                     | Includes CallForge logo and AI output processor description                                  |
| Sticky Note8                    | Sticky Note                | Labels Recurring Topics Processing block           | -                              | -                                     | "Recurring Topics"                                                                           |
| Sticky Note                     | Sticky Note                | Labels Actionable Insights Processing block        | -                              | -                                     | "Actionable Insights"                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add an **Execute Workflow Trigger** node.  
   - No special parameters; this node will receive AI sales call data as JSON input.

2. **Add Conditional Checks for Data Presence:**  
   - Add three **If** nodes named:  
     - "Check if Marketing Insight Data Found"  
     - "Check if Recurring Topics Found"  
     - "Check if Actionable Insights Data Found"  
   - Configure each to check if the respective array in `AIoutput` has length ≥ 1:  
     - Marketing Insights: `{{$json.AIoutput.MarketingInsights.length >= 1}}`  
     - Recurring Topics: `{{$json.AIoutput.RecurringTopics.length >= 1}}` (note: fix the original misconfiguration)  
     - Actionable Insights: `{{$json.AIoutput.ActionableInsights.length >= 1}}`

3. **Add Wait Nodes for Rate Limiting:**  
   - For each If node’s true branch, add a **Wait** node with 3 seconds delay:  
     - "Wait for rate limiting - Marketing Insights"  
     - "Wait for rate limiting - Recurring"  
     - "Wait for rate limiting - Actionable Insights"

4. **Split Arrays into Individual Items:**  
   - Add **SplitOut** nodes for each data type:  
     - "Split out Insights" splitting `AIoutput.MarketingInsights`  
     - "Split Out Recurring Topics" splitting `AIoutput.RecurringTopics`  
     - "Split Out Actionable Insights" splitting `AIoutput.ActionableInsights`

5. **Create Notion Pages for Each Item:**  
   - Add three **Notion** nodes configured to create new database pages:  
     - "Create Marketing Insight Data"  
       - Database: Marketing Insights (ID: `1395b6e0-c94f-802d-9a63-c524a1769699`)  
       - Properties:  
         - Name (title): `{{$json.Summary}}`  
         - Marketing Tags (multi-select): `{{$json.Tag}}`  
         - Sales Call Summaries (relation): `{{$json.notionData[0].id}}` from trigger  
         - Date Mentioned (date): `{{$json.metaData.started}}` from trigger  
       - Icon: 🎯  
       - Credentials: Notion API (e.g., "Notion david-internal")  
       - Retry enabled with 3s wait between tries.

     - "Create Recurring Topics Data"  
       - Database: Recurring Topics (ID: `17c5b6e0-c94f-80f4-9bf0-e52c7b0ef947`)  
       - Properties:  
         - Topic (title): `{{$json.Topic}}`  
         - Context (rich text): `{{$json.Context}}`  
         - Mentions (number): `{{$json.Mentions}}`  
         - Sales Call Summaries (relation): `{{$json.notionData[0].id}}` from trigger  
       - Icon: 🔁  
       - Credentials: Notion API (e.g., "Angelbot Notion")  
       - Retry enabled with 3s wait between tries.

     - "Create Actionable Insights Data"  
       - Database: Actionable Recommendations (ID: `17c5b6e0-c94f-809f-b5ee-e890f3ab3be9`)  
       - Properties:  
         - Topic (title): `{{$json.Topic}}`  
         - Rationale (rich text): `{{$json.Rationale}}`  
         - Recommendation Type (rich text): `{{$json.RecommendationType}}`  
         - Title (rich text): `{{$json.Title}}`  
         - Sales Call Summaries (relation): `{{$json.notionData[0].id}}` from trigger  
       - Icon: 🎬  
       - Credentials: Notion API (e.g., "Angelbot Notion")  
       - Retry enabled with 3s wait between tries.

6. **Aggregate Created Items:**  
   - Add **Aggregate** nodes for each category to bundle all created Notion pages into one object:  
     - "Bundle Marketing Insights Data to 1 object"  
     - "Bundle Recurring Topics Data to 1 object"  
     - "Bundle Actionable Insights Data to 1 object"  
   - Configure to aggregate all item data into a field named `tagdata`.

7. **Merge AI Response Context:**  
   - Add **Set** nodes to merge the original AI response JSON from the trigger into the aggregated data:  
     - "Merge Marketing Insights Thread"  
     - "Merge Recurring Topics Thread"  
     - "Merge Actionable Insights Thread"  
   - Assign `aiResponse` field with `{{$node["Execute Workflow Trigger"].json["aiResponse"]}}`.

8. **Connect Nodes:**  
   - Connect the trigger node to all three If nodes in parallel.  
   - Connect each If node’s true branch to its respective Wait node.  
   - Connect Wait nodes to SplitOut nodes.  
   - Connect SplitOut nodes to Notion create nodes.  
   - Connect Notion nodes to Aggregate nodes.  
   - Connect Aggregate nodes to Set nodes.

9. **Add Sticky Notes for Documentation:**  
   - Add sticky notes to label each block clearly:  
     - Input Reception  
     - Marketing Insights Processing  
     - Recurring Topics Processing  
     - Actionable Insights Processing

10. **Credential Setup:**  
    - Configure Notion API credentials for each Notion node as per your Notion integration setup.  
    - Ensure the databases exist with the expected schema and permissions.

11. **Testing:**  
    - Test the workflow by triggering with sample AI sales call data containing MarketingInsights, RecurringTopics, and ActionableInsights arrays.  
    - Verify entries are created correctly in Notion databases.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| CallForge branding and logo image used in Sticky Note7                                              | https://uploads.n8n.io/templates/callforgeshadow.png                                           |
| Workflow designed to integrate with Gong, Fireflies.ai, Otter.ai AI transcription tools            | Workflow description section                                                                   |
| Notion database IDs and credentials must be updated to match user’s Notion workspace and API keys  | Notion node configurations                                                                      |
| Rate limiting waits (3 seconds) are critical to avoid Notion API throttling errors                  | Wait nodes configuration                                                                       |
| Future integrations planned with Pipedrive and Salesforce (not yet implemented)                     | Sticky Note7 content                                                                            |
| Workflow supports n8n Cloud and self-hosted deployments                                            | Workflow description section                                                                   |

---

This document provides a comprehensive understanding of the CallForge workflow, enabling advanced users and AI agents to analyze, reproduce, and customize the automation for extracting marketing insights from AI-analyzed sales calls and storing them in Notion.