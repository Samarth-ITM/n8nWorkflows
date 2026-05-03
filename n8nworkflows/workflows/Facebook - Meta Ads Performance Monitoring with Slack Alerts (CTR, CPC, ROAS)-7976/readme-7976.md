Facebook / Meta Ads Performance Monitoring with Slack Alerts (CTR, CPC, ROAS)

https://n8nworkflows.xyz/workflows/facebook---meta-ads-performance-monitoring-with-slack-alerts--ctr--cpc--roas--7976


# Facebook / Meta Ads Performance Monitoring with Slack Alerts (CTR, CPC, ROAS)

### 1. Workflow Overview

This workflow automates the monitoring of Facebook/Meta Ads performance metrics such as CTR (Click-Through Rate), CPC (Cost Per Click), and ROAS (Return on Ad Spend). Upon manual trigger, it calculates a dynamic date range, fetches ad insights from the Facebook Graph API, processes and flattens the data, then sends a formatted alert message to a Slack channel. The workflow is designed for marketing teams who want real-time performance alerts to optimize ad campaigns rapidly.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Manual trigger initiation.
- **1.2 Date Range Calculation:** Computing the relevant reporting date range dynamically.
- **1.3 Data Retrieval:** Querying Facebook/Meta Ads API for ad performance insights.
- **1.4 Data Processing:** Flattening and formatting the retrieved insights.
- **1.5 Alerting:** Sending performance alerts to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Starts the workflow on manual execution.
- **Nodes Involved:** 
  - When clicking ‘Execute workflow’
- **Node Details:**
  - **Node:** When clicking ‘Execute workflow’  
    - **Type:** Manual Trigger  
    - **Role:** Entry point to manually initiate the workflow.  
    - **Configuration:** Default manual trigger with no parameters needed.  
    - **Inputs:** None (trigger node)  
    - **Outputs:** Connects to the "Edit Fields" node.  
    - **Edge Cases:** User needs to manually trigger; no scheduling or automatic start.  
    - **Version:** n8n v1.x compatible.

#### 1.2 Date Range Calculation

- **Overview:** Dynamically calculates the date range for which Facebook Ads data will be fetched.
- **Nodes Involved:** 
  - Edit Fields  
  - Calculate date range
- **Node Details:**
  - **Node:** Edit Fields  
    - **Type:** Set Node  
    - **Role:** Prepares or modifies fields/parameters before date calculation.  
    - **Configuration:** No parameters set explicitly; likely used as a placeholder or to reset/edit input data.  
    - **Inputs:** Receives trigger from manual node.  
    - **Outputs:** Connects to "Calculate date range".  
    - **Edge Cases:** If fields are not properly set, downstream nodes might fail or receive empty data.
  
  - **Node:** Calculate date range  
    - **Type:** Code Node (JavaScript)  
    - **Role:** Calculates the start and end dates for the Facebook Ads insights query.  
    - **Configuration:** Custom JavaScript code to determine date range dynamically, possibly relative to current date (e.g., last 7 days).  
    - **Key Expressions:** Uses JavaScript Date functions to generate ISO date strings.  
    - **Inputs:** From "Edit Fields".  
    - **Outputs:** To "Meta Ad Insights".  
    - **Edge Cases:**  
      - Date calculation errors if system time is incorrect.  
      - Improper formatting may cause API errors.  
    - **Version:** Requires n8n code node v2 or higher to support JavaScript ES6.

#### 1.3 Data Retrieval

- **Overview:** Queries Facebook/Meta Ads API to retrieve ad performance metrics for the calculated date range.
- **Nodes Involved:** 
  - Meta Ad Insights
- **Node Details:**
  - **Node:** Meta Ad Insights  
    - **Type:** Facebook Graph API Node  
    - **Role:** Fetches ads performance data such as CTR, CPC, ROAS from Facebook Ads API.  
    - **Configuration:**  
      - API endpoint configured to request insights data.  
      - Uses OAuth2 credentials for Facebook/Meta API authentication (credential setup required).  
      - Query parameters include dynamic date range from previous node.  
    - **Inputs:** From "Calculate date range".  
    - **Outputs:** To "Flatten Rows".  
    - **Edge Cases:**  
      - Authentication errors (invalid/expired OAuth tokens).  
      - API rate limits or throttling by Facebook.  
      - Invalid date ranges causing empty or error responses.  
    - **Version:** Compatible with n8n Facebook Graph API node v1.

#### 1.4 Data Processing

- **Overview:** Processes the raw Facebook Ads insights data, flattening nested JSON for easier consumption and alert formatting.
- **Nodes Involved:** 
  - Flatten Rows
- **Node Details:**
  - **Node:** Flatten Rows  
    - **Type:** Code Node (JavaScript)  
    - **Role:** Flattens and formats the nested insights data into a simplified structure for Slack alerts.  
    - **Configuration:** Custom code to iterate over Facebook Ads data, extract metrics like CTR, CPC, ROAS, and prepare human-readable output.  
    - **Inputs:** From "Meta Ad Insights".  
    - **Outputs:** To "Send an alert".  
    - **Edge Cases:**  
      - Unexpected data schema changes from Facebook API causing code errors.  
      - Empty or null data sets leading to empty alerts or skipped messages.  
    - **Version:** Requires n8n code node v2 or higher.

#### 1.5 Alerting

- **Overview:** Sends a formatted performance alert message to a designated Slack channel.
- **Nodes Involved:** 
  - Send an alert
- **Node Details:**
  - **Node:** Send an alert  
    - **Type:** Slack Node  
    - **Role:** Posts the processed ad performance data as an alert message to Slack.  
    - **Configuration:**  
      - Uses Slack webhook ID for authentication and message posting.  
      - Message content is dynamically generated from flattened data input.  
    - **Inputs:** From "Flatten Rows".  
    - **Outputs:** None (terminal node).  
    - **Edge Cases:**  
      - Slack webhook invalid or revoked causing message failures.  
      - Network issues preventing message delivery.  
    - **Version:** Uses Slack node v2.3 with webhook support.

---

### 3. Summary Table

| Node Name                  | Node Type                 | Functional Role                  | Input Node(s)               | Output Node(s)           | Sticky Note                    |
|----------------------------|---------------------------|--------------------------------|-----------------------------|---------------------------|-------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger            | Initiates the workflow manually | None                        | Edit Fields               |                               |
| Edit Fields                | Set Node                  | Prepares fields for date calc   | When clicking ‘Execute workflow’ | Calculate date range      |                               |
| Calculate date range       | Code Node                 | Calculates dynamic date range   | Edit Fields                 | Meta Ad Insights          |                               |
| Meta Ad Insights           | Facebook Graph API Node   | Fetches ad performance data     | Calculate date range        | Flatten Rows              |                               |
| Flatten Rows               | Code Node                 | Flattens and formats data       | Meta Ad Insights            | Send an alert             |                               |
| Send an alert              | Slack Node                | Sends alert message to Slack    | Flatten Rows                | None                      |                               |
| Sticky Note                | Sticky Note               | N/A                            | N/A                         | N/A                       |                               |
| Sticky Note1               | Sticky Note               | N/A                            | N/A                         | N/A                       |                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a "Manual Trigger" node named "When clicking ‘Execute workflow’".  
   - No parameters needed. This node serves as the workflow entry point.

2. **Add Set Node for Field Preparation**  
   - Add a "Set" node named "Edit Fields".  
   - No explicit parameters configured, used to prepare or reset data.  
   - Connect output of "When clicking ‘Execute workflow’" to this node.

3. **Add Code Node to Calculate Date Range**  
   - Create a "Code" node named "Calculate date range".  
   - Insert JavaScript to calculate start and end dates dynamically, e.g., last 7 days:  
     ```javascript
     const today = new Date();
     const endDate = today.toISOString().split('T')[0];
     const startDate = new Date(today.setDate(today.getDate() - 7)).toISOString().split('T')[0];
     return [{ json: { startDate, endDate } }];
     ```  
   - Connect output of "Edit Fields" to this node.

4. **Add Facebook Graph API Node to Fetch Ads Insights**  
   - Add a "Facebook Graph API" node named "Meta Ad Insights".  
   - Configure OAuth2 credentials for Facebook/Meta Ads API (ensure permissions for ads_insights).  
   - Set API endpoint to query ads insights (e.g., `/act_<AD_ACCOUNT_ID>/insights`).  
   - Use parameters to pass `startDate` and `endDate` from the previous node’s output.  
   - Connect output of "Calculate date range" to this node.

5. **Add Code Node to Flatten Ads Data**  
   - Add a "Code" node named "Flatten Rows".  
   - Write JavaScript to iterate over Facebook Ads insights data, extract key metrics (CTR, CPC, ROAS), and flatten nested structures into a simple JSON array for Slack.  
   - Connect output of "Meta Ad Insights" to this node.

6. **Add Slack Node to Send Alert**  
   - Add a "Slack" node named "Send an alert".  
   - Set up Slack credentials using Slack webhook URL.  
   - Configure the message to include dynamic data from "Flatten Rows" node (e.g., performance metrics summary).  
   - Connect output of "Flatten Rows" to this node.

7. **Save and Test**  
   - Save the workflow.  
   - Click "Execute workflow" manually to test data retrieval and Slack alert delivery.

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                            |
|------------------------------------------------------------------------------|------------------------------------------------------------|
| Workflow requires Facebook Marketing API OAuth2 credentials with ads insights permissions. | Facebook Developer Portal: https://developers.facebook.com/ |
| Slack webhook setup is needed for alert posting; create incoming webhook in Slack app. | Slack Incoming Webhooks: https://api.slack.com/messaging/webhooks |
| Facebook Ads insights API may have rate limits and quota restrictions.       | Facebook API Rate Limiting Docs: https://developers.facebook.com/docs/graph-api/overview/rate-limiting/ |
| This workflow runs manually and is intended for ad performance monitoring alerts. |                                                              |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, adhering strictly to content policies with no illegal, offensive, or protected elements. All handled data are legal and public.