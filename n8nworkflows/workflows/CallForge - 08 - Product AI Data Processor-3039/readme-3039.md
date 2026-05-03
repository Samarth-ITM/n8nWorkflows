CallForge - 08 - Product AI Data Processor

https://n8nworkflows.xyz/workflows/callforge---08---product-ai-data-processor-3039


# CallForge - 08 - Product AI Data Processor

### 1. Workflow Overview

This workflow, titled **CallForge - 08 - Product AI Data Processor**, is designed to automate the extraction, categorization, and storage of product-related insights from AI-analyzed sales call data. It targets product managers, engineering teams, and customer success teams who want to efficiently gather product feedback, AI/ML mentions, feature requests, and customer pain points from sales conversations. The workflow processes AI-generated sales call insights and organizes them into Notion databases for streamlined product intelligence and decision-making.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives AI-generated sales call data from an external workflow trigger.
- **1.2 Product Data Detection and Processing:** Checks for product feedback presence, processes and stores product feedback data in Notion.
- **1.3 AI Use Case Detection and Processing:** Checks for AI/ML-related mentions, processes and stores AI use case data in Notion.
- **1.4 AI Mention Summary Update:** Updates the original sales call record in Notion with AI-related summary data.
- **1.5 Rate Limiting and Data Aggregation:** Implements wait nodes to handle API rate limits and aggregates data objects before final storage.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block receives AI-generated sales call data from an external workflow and triggers the processing pipeline.

- **Nodes Involved:**  
  - Execute Workflow Trigger

- **Node Details:**  
  - **Execute Workflow Trigger**  
    - Type: `Execute Workflow Trigger` node  
    - Role: Entry point that receives AI sales call data from another workflow.  
    - Configuration: No parameters; it triggers when upstream workflow sends data.  
    - Inputs: None (trigger node)  
    - Outputs: Feeds data to three conditional checks for product data, AI use case data, and AI mentions.  
    - Edge Cases: Failure if upstream workflow does not send expected data structure; missing or malformed AI output JSON could cause downstream errors.

#### 2.2 Product Data Detection and Processing

- **Overview:**  
  This block detects if product feedback exists in the AI output, processes each feedback item, and stores structured product feedback data in Notion.

- **Nodes Involved:**  
  - Check if Product Data Found (If node)  
  - Wait for rate limiting - Product Data (Wait node)  
  - Split Out Product Data (Split Out node)  
  - Create Product Feedback Data Object (Notion node)  
  - Bundle Product Feedback Data to 1 object (Aggregate node)  
  - Merge Product Feedback Thread (Set node)  
  - Sticky Note4 (Labeling)

- **Node Details:**  
  - **Check if Product Data Found**  
    - Type: If node  
    - Role: Checks if the `AIoutput.ProductFeedback` array exists and contains at least one item.  
    - Condition: Array length of `AIoutput.ProductFeedback` >= 1  
    - Inputs: Data from Execute Workflow Trigger  
    - Outputs: True branch triggers product data processing; false branch skips.  
    - Edge Cases: If `AIoutput.ProductFeedback` is missing or not an array, condition fails; expression errors if path is invalid.

  - **Wait for rate limiting - Product Data**  
    - Type: Wait node  
    - Role: Delays processing by 3 seconds to avoid hitting Notion API rate limits.  
    - Inputs: True branch of product data check  
    - Outputs: Connects to Split Out Product Data node  
    - Edge Cases: None significant; ensures API compliance.

  - **Split Out Product Data**  
    - Type: Split Out node  
    - Role: Splits the `AIoutput.ProductFeedback` array into individual items for separate processing.  
    - Configuration: Splits on field `AIoutput.ProductFeedback`  
    - Inputs: From Wait node  
    - Outputs: Each feedback item sent to Create Product Feedback Data Object node  
    - Edge Cases: Empty arrays result in no output; malformed data could cause errors.

  - **Create Product Feedback Data Object**  
    - Type: Notion node  
    - Role: Creates a new page in the "Product Feedback" Notion database for each feedback item.  
    - Configuration:  
      - Title: From metadata title of the call  
      - Properties: Sentiment, Feedback text, Feedback Date, and relation to the sales call summary page  
      - Credentials: Uses "Notion david-internal" API credentials  
    - Inputs: Individual feedback items from Split Out node  
    - Outputs: Feeds into aggregation node  
    - Edge Cases: API failures, invalid property values, or credential issues.

  - **Bundle Product Feedback Data to 1 object**  
    - Type: Aggregate node  
    - Role: Aggregates all created product feedback pages into a single object for further processing or logging.  
    - Inputs: From Create Product Feedback Data Object  
    - Outputs: Connects to Merge Product Feedback Thread  
    - Edge Cases: No input items cause empty aggregation.

  - **Merge Product Feedback Thread**  
    - Type: Set node  
    - Role: Adds AI response metadata from the original trigger to the aggregated product feedback data.  
    - Inputs: From aggregation node or false branch of product data check (empty)  
    - Outputs: End of product feedback processing branch  
    - Edge Cases: Missing AI response data.

  - **Sticky Note4**  
    - Content: "## Product Data Processing"  
    - Role: Visual label grouping product data processing nodes.

#### 2.3 AI Use Case Detection and Processing

- **Overview:**  
  This block checks for AI/ML-related mentions in the call, processes AI use case data, and stores it in a dedicated Notion database.

- **Nodes Involved:**  
  - Check if AI Use Case Data Found (If node)  
  - Wait for rate limiting - AI Use Case (Wait node)  
  - Create Product Data Object1 (Notion node)  
  - Bundle AI Use Case Data to 1 object (Aggregate node)  
  - Merge AI Use Case Thread (Set node)  
  - Sticky Note6 (Labeling)  
  - Sticky Note (Labeling)

- **Node Details:**  
  - **Check if AI Use Case Data Found**  
    - Type: If node  
    - Role: Checks if `AIoutput.AI_ML_References.Exist` is true, indicating AI/ML mentions exist.  
    - Inputs: From Execute Workflow Trigger  
    - Outputs: True branch triggers AI use case processing; false branch skips.  
    - Edge Cases: Missing field or unexpected data types cause condition failure.

  - **Wait for rate limiting - AI Use Case**  
    - Type: Wait node  
    - Role: Delays processing by 3 seconds to respect Notion API rate limits.  
    - Inputs: True branch of AI use case check  
    - Outputs: Connects to Create Product Data Object1 node  
    - Edge Cases: None significant.

  - **Create Product Data Object1**  
    - Type: Notion node  
    - Role: Creates a new page in the "AI use-case database" Notion database with AI/ML reference details.  
    - Configuration:  
      - Title: From call metadata title  
      - Icon: 💬  
      - Properties: Company name, Department, Development status, Employees, Engagement status, Requires agents/chat/RAG, More info URL, Use case description  
      - Credentials: Uses "Angelbot Notion" API credentials  
    - Inputs: From Wait node  
    - Outputs: Feeds into aggregation node  
    - Edge Cases: API errors, invalid property values, credential issues.

  - **Bundle AI Use Case Data to 1 object**  
    - Type: Aggregate node  
    - Role: Aggregates all AI use case pages created into a single object.  
    - Inputs: From Create Product Data Object1  
    - Outputs: Connects to Merge AI Use Case Thread  
    - Edge Cases: Empty input results in empty aggregation.

  - **Merge AI Use Case Thread**  
    - Type: Set node  
    - Role: Adds AI response metadata from the original trigger to the aggregated AI use case data.  
    - Inputs: From aggregation node or false branch of AI use case check (empty)  
    - Outputs: End of AI use case processing branch  
    - Edge Cases: Missing AI response data.

  - **Sticky Note6**  
    - Content: "## Receives AI Data from other workflow"  
    - Role: Visual label near the trigger node.

  - **Sticky Note**  
    - Content: "## AI use Case"  
    - Role: Visual label grouping AI use case processing nodes.

#### 2.4 AI Mention Summary Update

- **Overview:**  
  This block updates the original sales call record in Notion with a summary of AI-related mentions detected during the call.

- **Nodes Involved:**  
  - Check if AI Mentioned On Call (If node)  
  - Update Call with AI Data Summary (Notion node)  
  - Sticky Note5 (Labeling)

- **Node Details:**  
  - **Check if AI Mentioned On Call**  
    - Type: If node  
    - Role: Checks if AI/ML references exist in the AI output (`AIoutput.AI_ML_References.Exist` is true).  
    - Inputs: From Execute Workflow Trigger  
    - Outputs: True branch triggers update; false branch skips.  
    - Edge Cases: Missing or malformed data causes condition failure.

  - **Update Call with AI Data Summary**  
    - Type: Notion node  
    - Role: Updates the existing sales call page in Notion with AI-related checkbox and summary text.  
    - Configuration:  
      - Page ID: From the first item in `notionData` array of the trigger data  
      - Properties updated:  
        - "AI Related" checkbox set to true/false based on AI mention existence  
        - "AI Summary" rich text set to AI/ML context string  
      - Credentials: Uses "Notion david-internal" API credentials  
    - Inputs: From If node  
    - Outputs: End of AI mention update branch  
    - Edge Cases: API update failures, invalid page ID, credential issues.

  - **Sticky Note5**  
    - Content: "## AI Mentioned on call"  
    - Role: Visual label grouping AI mention update nodes.

#### 2.5 Rate Limiting and Data Aggregation (Cross-Cutting)

- **Overview:**  
  This block ensures API rate limits are respected by introducing wait periods before Notion API calls and aggregates multiple data items into single objects for efficient processing.

- **Nodes Involved:**  
  - Wait for rate limiting - AI Use Case (Wait node)  
  - Wait for rate limiting - Product Data (Wait node)  
  - Bundle AI Use Case Data to 1 object (Aggregate node)  
  - Bundle Product Feedback Data to 1 object (Aggregate node)

- **Node Details:**  
  - **Wait Nodes**  
    - Purpose: Introduce 3-second delays before Notion API calls to avoid rate limiting.  
    - Inputs: Conditional checks for data presence  
    - Outputs: Proceed to Notion create nodes  
    - Edge Cases: None significant; ensures smooth API interaction.

  - **Aggregate Nodes**  
    - Purpose: Combine multiple created Notion pages into single data objects for downstream processing or logging.  
    - Inputs: From respective Notion create nodes  
    - Outputs: Feed into Set nodes that merge AI response metadata  
    - Edge Cases: Empty inputs result in empty aggregates.

---

### 3. Summary Table

| Node Name                        | Node Type                   | Functional Role                          | Input Node(s)                     | Output Node(s)                      | Sticky Note                                                                                     |
|---------------------------------|-----------------------------|----------------------------------------|----------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| Execute Workflow Trigger         | Execute Workflow Trigger     | Entry point receiving AI sales call data | None                             | Check if Product Data Found, Check if AI Use Case Data Found, Check if AI Mentioned On Call | ## Receives AI Data from other workflow                                                        |
| Check if Product Data Found      | If                          | Checks for presence of product feedback | Execute Workflow Trigger         | Wait for rate limiting - Product Data (true), Merge Product Feedback Thread (false)         | ## Product Data Processing                                                                     |
| Wait for rate limiting - Product Data | Wait                     | Delays to avoid Notion API rate limits | Check if Product Data Found      | Split Out Product Data             |                                                                                                |
| Split Out Product Data           | Split Out                   | Splits product feedback array into items | Wait for rate limiting - Product Data | Create Product Feedback Data Object |                                                                                                |
| Create Product Feedback Data Object | Notion                    | Creates Notion pages for product feedback | Split Out Product Data           | Bundle Product Feedback Data to 1 object |                                                                                                |
| Bundle Product Feedback Data to 1 object | Aggregate              | Aggregates product feedback pages       | Create Product Feedback Data Object | Merge Product Feedback Thread      |                                                                                                |
| Merge Product Feedback Thread    | Set                         | Merges AI response metadata with feedback data | Bundle Product Feedback Data to 1 object, Check if Product Data Found (false) | None                              |                                                                                                |
| Check if AI Use Case Data Found  | If                          | Checks for AI/ML mentions presence       | Execute Workflow Trigger         | Wait for rate limiting - AI Use Case (true), Merge AI Use Case Thread (false)               |                                                                                                |
| Wait for rate limiting - AI Use Case | Wait                      | Delays to avoid Notion API rate limits | Check if AI Use Case Data Found  | Create Product Data Object1        |                                                                                                |
| Create Product Data Object1      | Notion                      | Creates Notion pages for AI use cases    | Wait for rate limiting - AI Use Case | Bundle AI Use Case Data to 1 object |                                                                                                |
| Bundle AI Use Case Data to 1 object | Aggregate                 | Aggregates AI use case pages              | Create Product Data Object1      | Merge AI Use Case Thread           |                                                                                                |
| Merge AI Use Case Thread         | Set                         | Merges AI response metadata with AI use case data | Bundle AI Use Case Data to 1 object, Check if AI Use Case Data Found (false) | None                              |                                                                                                |
| Check if AI Mentioned On Call    | If                          | Checks if AI mentioned on call            | Execute Workflow Trigger         | Update Call with AI Data Summary (true), None (false)                                     | ## AI Mentioned on call                                                                        |
| Update Call with AI Data Summary | Notion                     | Updates sales call page with AI summary   | Check if AI Mentioned On Call    | None                             |                                                                                                |
| Sticky Note4                    | Sticky Note                 | Visual label for product data processing  | None                             | None                             | ## Product Data Processing                                                                     |
| Sticky Note6                    | Sticky Note                 | Visual label near trigger node             | None                             | None                             | ## Receives AI Data from other workflow                                                       |
| Sticky Note                     | Sticky Note                 | Visual label for AI use case processing    | None                             | None                             | ## AI use Case                                                                                |
| Sticky Note5                    | Sticky Note                 | Visual label for AI mention update         | None                             | None                             | ## AI Mentioned on call                                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add an **Execute Workflow Trigger** node named "Execute Workflow Trigger".  
   - No parameters needed. This node will receive AI sales call data from another workflow.

2. **Add Conditional Checks**  
   - Add an **If** node named "Check if Product Data Found".  
     - Condition: Check if `AIoutput.ProductFeedback` array length >= 1.  
     - Use expression: `={{ $json.AIoutput.ProductFeedback.length >= 1 }}`.  
   - Add an **If** node named "Check if AI Use Case Data Found".  
     - Condition: Check if `AIoutput.AI_ML_References.Exist` is true.  
     - Expression: `={{ $json.AIoutput.AI_ML_References.Exist === true }}`.  
   - Add an **If** node named "Check if AI Mentioned On Call".  
     - Same condition as AI Use Case Data Found.

3. **Connect Trigger to All Three If Nodes**  
   - Connect "Execute Workflow Trigger" main output to all three If nodes.

4. **Product Data Processing Branch**  
   - From "Check if Product Data Found" true output, add a **Wait** node named "Wait for rate limiting - Product Data" with 3 seconds delay.  
   - Connect Wait node to a **Split Out** node named "Split Out Product Data".  
     - Configure to split on field `AIoutput.ProductFeedback`.  
   - Connect Split Out node to a **Notion** node named "Create Product Feedback Data Object".  
     - Operation: Create database page in "Product Feedback" database.  
     - Title: `={{ $('Execute Workflow Trigger').item.json.metaData.title }}`  
     - Properties:  
       - Sentiment (multi-select): `={{ $json.Sentiment }}`  
       - Feedback (title): `={{ $json.Feedback }}`  
       - Feedback Date (date): `={{ $('Execute Workflow Trigger').item.json.metaData.started }}`  
       - Sales Call Summaries (relation): `={{ $('Execute Workflow Trigger').item.json.notionData[0].id }}`  
     - Credentials: Configure with your Notion API credentials.  
   - Connect Notion node to an **Aggregate** node named "Bundle Product Feedback Data to 1 object".  
     - Aggregate all item data into field "tagdata".  
   - Connect Aggregate node to a **Set** node named "Merge Product Feedback Thread".  
     - Add field "aiResponse" with value: `={{ $('Execute Workflow Trigger').item.json.aiResponse }}`.  
   - From "Check if Product Data Found" false output, connect directly to "Merge Product Feedback Thread" to handle empty case.

5. **AI Use Case Processing Branch**  
   - From "Check if AI Use Case Data Found" true output, add a **Wait** node named "Wait for rate limiting - AI Use Case" with 3 seconds delay.  
   - Connect Wait node to a **Notion** node named "Create Product Data Object1".  
     - Operation: Create database page in "AI use-case database".  
     - Title: `={{ $('Execute Workflow Trigger').item.json.metaData.title }}`  
     - Icon: 💬  
     - Properties:  
       - Company (title): `={{ $json.metaData.CompanyName }}`  
       - Department (multi-select): `={{ $json.AIoutput.AI_ML_References.Details.Department }}`  
       - Dev status (select): `={{ $json.AIoutput.AI_ML_References.Details.DevelopmentStatus }}`  
       - Employees (select): `={{ $json.sfOpp[0].Employees }}`  
       - Engagement with n8n (select): "Prospect" (static)  
       - Requires agents (checkbox): `={{ $json.AIoutput.AI_ML_References.Details.RequiresAgents }}`  
       - More info (url): `={{ $json.metaData.url }}`  
       - Requires RAG (checkbox): `={{ $json.AIoutput.AI_ML_References.Details.RequiresRAG }}`  
       - Requires chat (select): `={{ $json.AIoutput.AI_ML_References.Details.RequiresChat }}`  
       - Use case (rich text): `={{ $json.AIoutput.AI_ML_References.Context }}`  
     - Credentials: Configure with your Notion API credentials.  
   - Connect Notion node to an **Aggregate** node named "Bundle AI Use Case Data to 1 object".  
     - Aggregate all item data into field "tagdata".  
   - Connect Aggregate node to a **Set** node named "Merge AI Use Case Thread".  
     - Add field "aiResponse" with value: `={{ $('Execute Workflow Trigger').item.json.aiResponse }}`.  
   - From "Check if AI Use Case Data Found" false output, connect directly to "Merge AI Use Case Thread" to handle empty case.

6. **AI Mention Summary Update Branch**  
   - From "Check if AI Mentioned On Call" true output, add a **Notion** node named "Update Call with AI Data Summary".  
     - Operation: Update database page.  
     - Page ID: `={{ $('Execute Workflow Trigger').item.json.notionData[0].id }}`  
     - Properties to update:  
       - AI Related (checkbox): `={{ $json.AIoutput.AI_ML_References.Exist }}`  
       - AI Summary (rich text): `={{ $json.AIoutput.AI_ML_References.Context }}`  
     - Credentials: Configure with your Notion API credentials.

7. **Add Sticky Notes for Clarity**  
   - Add sticky notes with the following content and place them near relevant nodes for documentation clarity:  
     - "## Receives AI Data from other workflow" near the trigger node.  
     - "## Product Data Processing" near product data nodes.  
     - "## AI use Case" near AI use case nodes.  
     - "## AI Mentioned on call" near AI mention update nodes.  
     - Optionally add the CallForge logo and description as a sticky note for branding.

8. **Credential Setup**  
   - Configure Notion API credentials for each Notion node with appropriate access to the target databases.  
   - Ensure the external workflow sending data to the Execute Workflow Trigger node is properly configured.

9. **Testing**  
   - Test the workflow with sample AI sales call data containing product feedback and AI mentions.  
   - Verify Notion pages are created and updated correctly.  
   - Monitor for rate limiting or API errors and adjust wait times if necessary.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| CallForge is an AI Gong Sales Call Processor that extracts important information for multiple departments from sales calls. | ![Callforge Logo](https://uploads.n8n.io/templates/callforgeshadow.png)                             |
| This workflow automates product feedback tracking and AI mention detection, storing insights in Notion for product teams. | Workflow description and use case overview                                                         |
| To extend functionality, integrate with Slack, email, Jira, Trello, or Asana for notifications and ticket creation. | Customization suggestions                                                                           |
| Ensure Notion databases for Product Feedback and AI Use Cases are set up with matching property schemas. | Setup guide                                                                                        |
| Use the "Angelbot Notion" and "Notion david-internal" credentials for respective Notion nodes.  | Credential configuration                                                                            |

---

This documentation provides a detailed, structured reference for the **CallForge - 08 - Product AI Data Processor** workflow, enabling users and AI agents to understand, reproduce, and modify the workflow confidently.