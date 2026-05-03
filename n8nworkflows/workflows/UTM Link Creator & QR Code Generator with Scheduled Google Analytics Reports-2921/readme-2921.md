UTM Link Creator & QR Code Generator with Scheduled Google Analytics Reports

https://n8nworkflows.xyz/workflows/utm-link-creator---qr-code-generator-with-scheduled-google-analytics-reports-2921


# UTM Link Creator & QR Code Generator with Scheduled Google Analytics Reports

---

### 1. Workflow Overview

This workflow automates the creation, storage, and performance tracking of UTM-tagged marketing links combined with QR code generation and scheduled Google Analytics reporting. It is designed for marketers who want to efficiently generate trackable URLs, convert them into QR codes for offline or digital campaigns, and receive automated weekly performance summaries based on Google Analytics data.

The workflow is logically divided into three main blocks:

- **1.1 UTM Link Creation & Storage:** Receives input parameters, constructs UTM-tagged URLs, stores them in Airtable, and generates corresponding QR codes.
- **1.2 Scheduled Google Analytics Reporting:** Periodically triggers data retrieval from Google Analytics to analyze campaign performance based on UTM parameters.
- **1.3 AI-Powered Analytics Summary & Notification:** Processes the analytics data with an AI agent to generate an executive summary and emails the report to the marketing manager.

---

### 2. Block-by-Block Analysis

#### 2.1 UTM Link Creation & Storage

**Overview:**  
This block accepts campaign parameters, constructs UTM-tagged URLs, stores them in Airtable, and generates QR codes for each URL to facilitate easy sharing and tracking.

**Nodes Involved:**  
- Create UTM Link & Send To Database (Manual Trigger)  
- Set UTM Parameters For Link (Set)  
- Create UTM Link With Parameters (Code)  
- Submit UTM Link To Database (Airtable)  
- Create QR Code With Submitted QR Link (HTTP Request)  
- Sticky Note (two notes related to this block)

**Node Details:**

- **Create UTM Link & Send To Database**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for manual initiation of UTM link creation.  
  - *Configuration:* No parameters; triggers workflow execution manually.  
  - *Connections:* Outputs to "Set UTM Parameters For Link".  
  - *Edge Cases:* None specific; manual trigger requires user action.

- **Set UTM Parameters For Link**  
  - *Type:* Set  
  - *Role:* Defines and assigns UTM parameters and base URL for link creation.  
  - *Configuration:* Hardcoded values for parameters such as website URL, campaign ID, source, medium, name, and term.  
  - *Key Variables:*  
    - `website_url`: "https://ecconcretecoating.com/"  
    - `campaign_id`: "12246"  
    - `campaign_source`: "google"  
    - `campaign_medium`: "display"  
    - `campaign_name`: "summerfun"  
    - `campaign_term`: "conretecoating" (note: typo in "conretecoating")  
  - *Connections:* Outputs to "Create UTM Link With Parameters".  
  - *Edge Cases:* Hardcoded values limit flexibility; consider dynamic input for scalability.

- **Create UTM Link With Parameters**  
  - *Type:* Code (JavaScript)  
  - *Role:* Constructs the full UTM-tagged URL by appending parameters to the base URL.  
  - *Configuration:* Uses JavaScript to concatenate parameters into a URL string, adding `utm_source`, `utm_medium`, `utm_campaign`, `utm_term`, and `utm_content` (mapped to campaign ID).  
  - *Key Expression:*  
    ```js
    const utmUrl = `${website_url}?utm_source=${campaign_source}&utm_medium=${campaign_medium}&utm_campaign=${campaign_name}&utm_term=${campaign_term}&utm_content=${campaign_id}`;
    ```  
  - *Connections:* Outputs to both "Submit UTM Link To Database" and "Create QR Code With Submitted QR Link".  
  - *Edge Cases:* Assumes all parameters are present; missing or empty parameters may produce malformed URLs.

- **Submit UTM Link To Database**  
  - *Type:* Airtable  
  - *Role:* Upserts the generated UTM URL into a specified Airtable base and table for record-keeping.  
  - *Configuration:*  
    - Base: "appIXd8a8JeB9bPaL"  
    - Table: "tblXyFxXMHraieGCa" (UTM_URL)  
    - Column mapping: Airtable "URL" field receives `utmUrl` value.  
    - Operation: Upsert based on record ID.  
  - *Credentials:* Airtable Personal Access Token.  
  - *Connections:* None (end of this branch).  
  - *Edge Cases:* Airtable API errors (auth failure, rate limits), data mismatch, or missing fields.

- **Create QR Code With Submitted QR Link**  
  - *Type:* HTTP Request  
  - *Role:* Generates a QR code image URL for the UTM link using the QuickChart API.  
  - *Configuration:*  
    - URL template: `https://quickchart.io/qr?text={{ $json.utmUrl }}&size=300&margin=10&ecLevel=H&dark=000000&light=FFFFFF`  
    - Method: GET (default)  
  - *Connections:* None (end of this branch).  
  - *Edge Cases:* API downtime, malformed URLs, or rate limits.

- **Sticky Notes:**  
  - One note explains the input requirements for UTM parameters and the overall purpose of creating marketing links with UTM tags and QR codes.  
  - Another note describes the code node’s function and Airtable setup recommendations.

---

#### 2.2 Scheduled Google Analytics Reporting

**Overview:**  
This block triggers every 7 days to retrieve Google Analytics 4 data related to UTM parameters, enabling performance tracking of marketing campaigns.

**Nodes Involved:**  
- Schedule Google Analytics Report To Marketing Manager (Schedule Trigger)  
- Google Analytics (Google Analytics Tool)  
- Google Analytics Data Analysis Agent (Langchain Agent)  
- Sticky Note2 (related to scheduling and reporting)

**Node Details:**

- **Schedule Google Analytics Report To Marketing Manager**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers the workflow on a recurring schedule (every 7 days).  
  - *Configuration:* Interval set to 7 days (weekly).  
  - *Connections:* Outputs to "Google Analytics Data Analysis Agent".  
  - *Edge Cases:* Scheduling misconfiguration, missed triggers if n8n instance is down.

- **Google Analytics**  
  - *Type:* Google Analytics Tool  
  - *Role:* Queries GA4 Data API for metrics and dimensions related to UTM campaigns.  
  - *Configuration:*  
    - Property ID: 404306108 (East Coast Concrete Coating)  
    - Metrics: Sessions  
    - Dimensions: Source/Medium  
  - *Credentials:* Google Analytics OAuth2.  
  - *Connections:* Outputs data to "Google Analytics Data Analysis Agent" via AI tool connection.  
  - *Edge Cases:* API quota limits, auth token expiration, incorrect property ID, missing metrics/dimensions.

- **Sticky Note2:**  
  - Describes the scheduling of Google Analytics reports, tracking UTM link performance, and suggests updating metrics as needed.

---

#### 2.3 AI-Powered Analytics Summary & Notification

**Overview:**  
This block uses an AI agent to analyze the retrieved Google Analytics data, generate an executive summary report, and send it via email to the marketing manager.

**Nodes Involved:**  
- Google Analytics Data Analysis Agent (Langchain Agent)  
- OpenAI Chat Model1 (OpenAI Language Model)  
- Window Buffer Memory (Langchain Memory)  
- Send Summary Report To Marketing Manager (Gmail)  

**Node Details:**

- **Google Analytics Data Analysis Agent**  
  - *Type:* Langchain Agent  
  - *Role:* Processes GA data and generates a structured executive summary report.  
  - *Configuration:*  
    - System message instructs the AI to create a professional, clear summary with sections: Overview, KPIs, Trends & Insights, Opportunities & Recommendations, Conclusion.  
    - Input text is the timestamp from the GA data.  
  - *Connections:* Outputs to "Send Summary Report To Marketing Manager".  
  - *Edge Cases:* AI model errors, incomplete data input, rate limits.

- **OpenAI Chat Model1**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Provides the underlying GPT-4o-mini language model for the AI agent.  
  - *Configuration:* Model set to "gpt-4o-mini".  
  - *Credentials:* OpenAI API key.  
  - *Connections:* Connected as AI language model resource for the agent.  
  - *Edge Cases:* API key issues, rate limits, model availability.

- **Window Buffer Memory**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Maintains conversational context for the AI agent, keyed by timestamp.  
  - *Configuration:* Context window length set to 200 tokens.  
  - *Connections:* Connected as AI memory resource for the agent.  
  - *Edge Cases:* Memory overflow, session key conflicts.

- **Send Summary Report To Marketing Manager**  
  - *Type:* Gmail  
  - *Role:* Sends the AI-generated summary report via email.  
  - *Configuration:*  
    - Recipient: john@marketingcanopy.com  
    - Subject: "Google Analytics Metrics Summary Report"  
    - Message body: AI agent output.  
  - *Credentials:* Gmail OAuth2.  
  - *Connections:* None (end node).  
  - *Edge Cases:* Email sending failures, auth token expiration.

---

### 3. Summary Table

| Node Name                              | Node Type                        | Functional Role                                | Input Node(s)                      | Output Node(s)                                   | Sticky Note                                                                                         |
|--------------------------------------|---------------------------------|-----------------------------------------------|----------------------------------|-------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Create UTM Link & Send To Database   | Manual Trigger                  | Entry point to start UTM link creation        | -                                | Set UTM Parameters For Link                      | Create a marketing link with UTM parameters. Easily store in database and have QR code created and ready as well. Type in requirements: website URL, campaign id, campaign source, campaign medium, campaign name, campaign term |
| Set UTM Parameters For Link          | Set                            | Defines UTM parameters for link construction  | Create UTM Link & Send To Database | Create UTM Link With Parameters                  |                                                                                                   |
| Create UTM Link With Parameters      | Code                           | Builds UTM-tagged URL from parameters          | Set UTM Parameters For Link       | Submit UTM Link To Database, Create QR Code With Submitted QR Link | Code node creates the URL with UTM parameters. It then sends to your Airtable database to store for records. It also creates a QR code with the embedded link to be used for materials. Sample Airtable Setup: -Website Link UTM column |
| Submit UTM Link To Database          | Airtable                       | Stores UTM link in Airtable base               | Create UTM Link With Parameters   | -                                               |                                                                                                   |
| Create QR Code With Submitted QR Link | HTTP Request                  | Generates QR code image URL for UTM link      | Create UTM Link With Parameters   | -                                               |                                                                                                   |
| Schedule Google Analytics Report To Marketing Manager | Schedule Trigger            | Triggers scheduled GA report every 7 days     | -                                | Google Analytics Data Analysis Agent             | Schedule a Google Analytics Reports with Medium/Source to track UTM link performance. Update the reporting fields to fit your business needs. You can track traffic, conversions and other engagement metrics. *Sample Google Report Metrics: Sessions. Update metrics as needed. |
| Google Analytics                     | Google Analytics Tool           | Retrieves GA4 metrics and dimensions           | -                                | Google Analytics Data Analysis Agent (via AI tool) |                                                                                                   |
| Google Analytics Data Analysis Agent | Langchain Agent                | Analyzes GA data and generates executive summary | Schedule Google Analytics Report To Marketing Manager | Send Summary Report To Marketing Manager         |                                                                                                   |
| OpenAI Chat Model1                   | Langchain OpenAI Chat Model    | Provides GPT-4o-mini language model for AI agent | -                                | Google Analytics Data Analysis Agent (AI languageModel) |                                                                                                   |
| Window Buffer Memory                 | Langchain Memory Buffer Window | Maintains AI conversational context            | -                                | Google Analytics Data Analysis Agent (AI memory) |                                                                                                   |
| Send Summary Report To Marketing Manager | Gmail                       | Sends AI-generated summary report via email   | Google Analytics Data Analysis Agent | -                                               |                                                                                                   |
| Sticky Note                         | Sticky Note                    | Notes on UTM link creation input requirements  | -                                | -                                               | Create a marketing link with UTM parameters. Easily store in database and have QR code created and ready as well. Type in requirements: website URL, campaign id, campaign source, campaign medium, campaign name, campaign term |
| Sticky Note1                        | Sticky Note                    | Notes on code node and Airtable setup           | -                                | -                                               | Code node creates the URL with UTM parameters. It then sends to your Airtable database to store for records. It also creates a QR code with the embedded link to be used for materials. Sample Airtable Setup: -Website Link UTM column |
| Sticky Note2                        | Sticky Note                    | Notes on scheduling Google Analytics reports    | -                                | -                                               | Schedule a Google Analytics Reports with Medium/Source to track UTM link performance. Update the reporting fields to fit your business needs. You can track traffic, conversions and other engagement metrics. *Sample Google Report Metrics: Sessions. Update metrics as needed. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "Create UTM Link & Send To Database"  
   - Type: Manual Trigger  
   - Purpose: Entry point for manual UTM link creation.

2. **Add Set Node**  
   - Name: "Set UTM Parameters For Link"  
   - Type: Set  
   - Configure fields:  
     - `website_url`: "https://ecconcretecoating.com/"  
     - `campaign_id`: "12246"  
     - `campaign_source`: "google"  
     - `campaign_medium`: "display"  
     - `campaign_name`: "summerfun"  
     - `campaign_term`: "conretecoating" (note: fix typo if needed)  
   - Connect output of Manual Trigger to this node.

3. **Add Code Node**  
   - Name: "Create UTM Link With Parameters"  
   - Type: Code (JavaScript)  
   - Paste the following code:  
     ```js
     const items = $input.all();
     const updatedItems = items.map((item) => {
       const utmUrl = `${item.json.website_url}?utm_source=${item.json.campaign_source}&utm_medium=${item.json.campaign_medium}&utm_campaign=${item.json.campaign_name}&utm_term=${item.json.campaign_term}&utm_content=${item.json.campaign_id}`;
       item.json.utmUrl = utmUrl;
       return item;
     });
     return updatedItems;
     ```  
   - Connect output of Set node to this node.

4. **Add Airtable Node**  
   - Name: "Submit UTM Link To Database"  
   - Type: Airtable  
   - Credentials: Configure Airtable Personal Access Token.  
   - Configure:  
     - Base: Select your Airtable base (e.g., "appIXd8a8JeB9bPaL")  
     - Table: Select table for UTM URLs (e.g., "tblXyFxXMHraieGCa")  
     - Operation: Upsert  
     - Map field "URL" to `{{ $json.utmUrl }}`  
   - Connect output of Code node to this node.

5. **Add HTTP Request Node**  
   - Name: "Create QR Code With Submitted QR Link"  
   - Type: HTTP Request  
   - Configure:  
     - Method: GET  
     - URL: `https://quickchart.io/qr?text={{ $json.utmUrl }}&size=300&margin=10&ecLevel=H&dark=000000&light=FFFFFF`  
   - Connect output of Code node to this node.

6. **Add Schedule Trigger Node**  
   - Name: "Schedule Google Analytics Report To Marketing Manager"  
   - Type: Schedule Trigger  
   - Configure: Set interval to every 7 days (weekly).  
   - This node triggers the analytics reporting flow.

7. **Add Google Analytics Node**  
   - Name: "Google Analytics"  
   - Type: Google Analytics Tool  
   - Credentials: Configure Google Analytics OAuth2 credentials.  
   - Configure:  
     - Property ID: Your GA4 property ID (e.g., "404306108")  
     - Metrics: Add "sessions" or other desired metrics.  
     - Dimensions: Add "sourceMedium" or other relevant dimensions.  
   - Connect output of Schedule Trigger to this node.

8. **Add Langchain Agent Node**  
   - Name: "Google Analytics Data Analysis Agent"  
   - Type: Langchain Agent  
   - Configure:  
     - Text input: `{{ $json.timestamp }}` or relevant GA data.  
     - System message: Provide detailed instructions for executive summary generation (as in the original workflow).  
   - Connect output of Schedule Trigger to this node.

9. **Add Langchain OpenAI Chat Model Node**  
   - Name: "OpenAI Chat Model1"  
   - Type: Langchain OpenAI Chat Model  
   - Credentials: Configure OpenAI API key.  
   - Configure model: "gpt-4o-mini" or preferred GPT-4 variant.  
   - Connect as AI language model resource to the Langchain Agent node.

10. **Add Langchain Memory Node**  
    - Name: "Window Buffer Memory"  
    - Type: Langchain Memory Buffer Window  
    - Configure:  
      - Session key: `{{ $json.timestamp }}`  
      - Context window length: 200 tokens  
    - Connect as AI memory resource to the Langchain Agent node.

11. **Add Gmail Node**  
    - Name: "Send Summary Report To Marketing Manager"  
    - Type: Gmail  
    - Credentials: Configure Gmail OAuth2 credentials.  
    - Configure:  
      - Recipient: "john@marketingcanopy.com" (replace as needed)  
      - Subject: "Google Analytics Metrics Summary Report"  
      - Message: `{{ $json.output }}` (output from Langchain Agent)  
    - Connect output of Langchain Agent node to this node.

12. **Connect Google Analytics Node output to Langchain Agent**  
    - The GA data output should be passed to the Langchain Agent node for analysis.

13. **Verify all connections and credentials**  
    - Ensure all API keys and OAuth2 credentials are properly set up and tested.

14. **Optional: Add Sticky Notes**  
    - Add notes to document input requirements, code logic, and scheduling details for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Google Analytics Data API Overview for scheduling performance tracking.                                              | https://developers.google.com/analytics/devguides/reporting/data/v1                                             |
| Airtable API Authentication Guide for setting up API keys and access.                                                | https://www.airtable.com/developers/web/api/authentication                                                      |
| Airtable Web API Documentation for base and table setup.                                                            | https://www.airtable.com/developers/web/api                                                                      |
| QR Code generation using QuickChart API (https://quickchart.io/documentation/qr-codes/)                              | https://quickchart.io/documentation/qr-codes/                                                                     |
| AI agent system message instructs generation of executive summaries tailored for marketing managers and executives. | Embedded in Langchain Agent node configuration                                                                    |
| Workflow designed to automate marketing campaign tracking, QR code generation, and scheduled reporting.               | Workflow purpose and benefits section in the initial description                                                 |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and modifying the "UTM Link Creator & QR Code Generator with Scheduled Google Analytics Reports" workflow in n8n. It covers all nodes, their roles, configurations, and integration points, enabling advanced users and AI agents to work effectively with the workflow.