Marketing Analytics Reports with Google Analytics, Sheets, Slides & Email Alerts

https://n8nworkflows.xyz/workflows/marketing-analytics-reports-with-google-analytics--sheets--slides---email-alerts-10715


# Marketing Analytics Reports with Google Analytics, Sheets, Slides & Email Alerts

---
### 1. Workflow Overview

This workflow automates the generation and distribution of weekly and monthly marketing analytics reports by integrating Google Analytics, advertising platform APIs, CRM data, Google Sheets, Google Slides, Gmail, and Slack. It targets marketing teams, data analysts, and business owners who require consistent, timely insights into key performance indicators (KPIs) such as Monthly Active Users (MAU), Customer Acquisition Cost (CAC), Lifetime Value (LTV), and conversion rates.

The workflow is logically divided into the following blocks:

- **1.1 Schedule Trigger:** Initiates the workflow on a weekly and monthly basis.
- **1.2 Configuration Setup:** Loads essential configuration parameters such as API endpoints, spreadsheet and presentation IDs, recipients, and reporting period.
- **1.3 Data Retrieval:** Fetches data from Google Analytics, the advertising platform, and the CRM system.
- **1.4 KPI Calculation:** Processes raw data to compute business-critical KPIs.
- **1.5 Data Logging:** Stores KPI data into a Google Sheets document for historical tracking.
- **1.6 Report Generation:** Creates a Google Slides presentation as the visual report.
- **1.7 Error Handling and Notifications:** Checks for errors and sends notifications via Slack if any occur.
- **1.8 Report Distribution:** Emails a summary of the KPIs with a link to the full Google Slides report.

---

### 2. Block-by-Block Analysis

#### 1.1 Schedule Trigger

- **Overview:** Triggers the workflow every Monday at 09:00 for weekly reports and on the first day of each month at 09:00 for monthly reports.
- **Nodes Involved:** `Weekly/Monthly Schedule`
- **Node Details:**
  - **Type:** Schedule Trigger
  - **Configuration:** 
    - Weekly interval: Mondays at 9 AM
    - Monthly interval: On the first day of the month at 9 AM
  - **Expressions:** None
  - **Inputs:** None (trigger node)
  - **Outputs:** Triggers `Workflow Configuration`
  - **Edge Cases:** Misconfiguration can lead to missed or duplicate triggers. Timezone considerations should be verified.
  - **Version:** 1.2

#### 1.2 Configuration Setup

- **Overview:** Sets static workflow parameters such as API URLs, Google resource IDs, email recipients, Slack channel, and reporting period.
- **Nodes Involved:** `Workflow Configuration`
- **Node Details:**
  - **Type:** Set
  - **Configuration:** Assigns multiple string and number variables:
    - `gaPropertyId`: Google Analytics Property ID placeholder
    - `adPlatformApiUrl`: Ad platform API endpoint placeholder
    - `crmApiUrl`: CRM API endpoint placeholder
    - `reportSpreadsheetId`: Google Sheets ID placeholder
    - `slidesTemplateId`: Google Slides template ID placeholder
    - `reportRecipients`: Comma-separated email addresses placeholder
    - `slackChannel`: Slack channel ID placeholder
    - `reportPeriodDays`: Number of days to look back (default 7)
  - **Expressions:** None, static assignment with placeholders
  - **Inputs:** From `Weekly/Monthly Schedule`
  - **Outputs:** Triggers data retrieval nodes
  - **Edge Cases:** Missing or incorrect values can cause API failures or incorrect data periods.
  - **Version:** 3.4

#### 1.3 Data Retrieval

- **Overview:** Collects analytics, advertising spend, and CRM data for the reporting period.
- **Nodes Involved:** `Get Google Analytics Data`, `Get Ad Platform Data`, `Get CRM Data`
- **Node Details:**

  - **Get Google Analytics Data**
    - **Type:** Google Analytics
    - **Configuration:**
      - Property ID dynamically set from `Workflow Configuration.gaPropertyId`
      - Date range: Custom from `reportPeriodDays` ago to yesterday
      - Metrics: active1DayUsers, sessions, screenPageViews, conversions
      - Dimensions: default (empty)
    - **Expressions:** Dates and property ID dynamically calculated using current date and configuration node
    - **Inputs:** From `Workflow Configuration`
    - **Outputs:** To `Calculate KPIs`
    - **Edge Cases:** API authentication failures, empty results, or quota limits.
    - **Version:** 2

  - **Get Ad Platform Data**
    - **Type:** HTTP Request
    - **Configuration:**
      - URL from `Workflow Configuration.adPlatformApiUrl`
      - Query parameters: `start_date`, `end_date` set dynamically
      - Authentication: uses predefined credentials (configured outside workflow)
    - **Inputs:** From `Workflow Configuration`
    - **Outputs:** To `Calculate KPIs`
    - **Edge Cases:** Network errors, auth failures, API changes.
    - **Version:** 4.3
  
  - **Get CRM Data**
    - **Type:** HTTP Request
    - **Configuration:**
      - URL from `Workflow Configuration.crmApiUrl`
      - Query parameters: `start_date`, `end_date` set dynamically
      - Authentication: predefined credentials
    - **Inputs:** From `Workflow Configuration`
    - **Outputs:** To `Calculate KPIs`
    - **Edge Cases:** Similar to Ad Platform node; also, potential data inconsistency.
    - **Version:** 4.3

#### 1.4 KPI Calculation

- **Overview:** Processes data from the three sources to compute KPIs including MAU, CAC, LTV, LTV:CAC ratio, conversion rate, and others.
- **Nodes Involved:** `Calculate KPIs (MAU, LTV, CAC)`
- **Node Details:**
  - **Type:** Code (JavaScript)
  - **Configuration:**
    - Reads JSON from Google Analytics, Ad Platform, and CRM nodes.
    - Calculates:
      - MAU from activeUsers
      - CAC = totalAdSpend / newCustomers
      - LTV = totalRevenue / totalCustomers
      - LTV:CAC ratio
      - Conversion rate = conversions / sessions * 100
    - Returns an object with calculated KPI fields and reporting period dates.
  - **Expressions/Variables:** Uses `$now` for dates, accesses previous nodes by name.
  - **Inputs:** From `Get Google Analytics Data`, `Get Ad Platform Data`, `Get CRM Data`
  - **Outputs:** To `Write Data to Google Sheets`
  - **Edge Cases:** Division by zero handled by defaulting denominators to 1; missing data may cause zero or inaccurate KPIs; expression errors if input data is malformed.
  - **Version:** 2

#### 1.5 Data Logging

- **Overview:** Appends or updates the KPI data in a Google Sheets document to maintain historical records.
- **Nodes Involved:** `Write Data to Google Sheets`
- **Node Details:**
  - **Type:** Google Sheets
  - **Configuration:**
    - Operation: Append or update rows
    - Sheet name: "KPI Data"
    - Document ID: from `Workflow Configuration.reportSpreadsheetId`
    - Columns mapped automatically based on input JSON keys
  - **Inputs:** From `Calculate KPIs (MAU, LTV, CAC)`
  - **Outputs:** To `Create Google Slides Report`
  - **Edge Cases:** API quota, permission issues, spreadsheet access errors.
  - **Version:** 4.7
  - **Credential:** Google Sheets OAuth2 required

#### 1.6 Report Generation

- **Overview:** Generates a Google Slides presentation titled with the current date to serve as the visual analytics report.
- **Nodes Involved:** `Create Google Slides Report`
- **Node Details:**
  - **Type:** Google Slides
  - **Configuration:**
    - Title dynamically set as: "Analytics Report - YYYY-MM-DD"
    - Uses a Google Slides template ID (configured in `Workflow Configuration`, but template ID usage is implied, not explicit in node parameters)
  - **Inputs:** From `Write Data to Google Sheets`
  - **Outputs:** To `Check for Errors`
  - **Edge Cases:** Template ID incorrect/missing, API errors.
  - **Version:** 2
  - **Credential:** Google Slides OAuth2 (assumed to be set)

#### 1.7 Error Handling and Notifications

- **Overview:** Checks for errors in the report generation process and routes flow accordingly; sends Slack notifications on failure.
- **Nodes Involved:** `Check for Errors`, `Send Error Notification`
- **Node Details:**

  - **Check for Errors**
    - **Type:** If
    - **Configuration:**
      - Condition: Checks if `error` field exists in JSON; if no error, proceeds to email sending; else triggers error notification.
    - **Inputs:** From `Create Google Slides Report`
    - **Outputs:** 
      - True branch: `Send Report Email`
      - False branch: `Send Error Notification`
    - **Edge Cases:** If error field is not properly set by previous node, false positives/negatives possible.
    - **Version:** 2.2

  - **Send Error Notification**
    - **Type:** Slack
    - **Configuration:**
      - Message includes error text from JSON or "Unknown error occurred"
      - Sends message to Slack channel ID from `Workflow Configuration.slackChannel`
      - OAuth2 authentication used
    - **Inputs:** From `Check for Errors` false output
    - **Outputs:** None (end node)
    - **Edge Cases:** Slack API failures, invalid channel ID, auth errors.
    - **Version:** 2.3
    - **Credential:** Slack OAuth2 required

#### 1.8 Report Distribution

- **Overview:** Sends an email summarizing the KPIs with a link to the generated Google Slides presentation.
- **Nodes Involved:** `Send Report Email`
- **Node Details:**
  - **Type:** Gmail
  - **Configuration:**
    - Recipients from `Workflow Configuration.reportRecipients`
    - Subject: "Weekly/Monthly Analytics Report - YYYY-MM-DD"
    - HTML message body contains:
      - Reporting period
      - KPI summary (MAU, CAC, LTV, LTV:CAC, conversion rate)
      - Additional metrics (sessions, conversions, new customers, ad spend, revenue)
      - Link to Google Slides presentation by referencing `Create Google Slides Report.presentationId`
    - Attachments: configured for binary attachments though none detailed in JSON
    - OAuth2 authentication
  - **Inputs:** From `Check for Errors` true output
  - **Outputs:** None (end node)
  - **Edge Cases:** Email delivery failures, invalid recipient addresses, Gmail quota limits.
  - **Version:** 2.1
  - **Credential:** Gmail OAuth2 required

---

### 3. Summary Table

| Node Name                   | Node Type           | Functional Role                               | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                         |
|-----------------------------|---------------------|-----------------------------------------------|----------------------------|-------------------------------|---------------------------------------------------------------------------------------------------|
| Weekly/Monthly Schedule      | Schedule Trigger    | Starts workflow on weekly/monthly schedule    | None                       | Workflow Configuration          |                                                                                                   |
| Workflow Configuration       | Set                 | Holds all configurable parameters             | Weekly/Monthly Schedule    | Get Google Analytics Data, Get Ad Platform Data, Get CRM Data |                                                                                                   |
| Get Google Analytics Data    | Google Analytics    | Retrieves analytics data from GA               | Workflow Configuration     | Calculate KPIs (MAU, LTV, CAC) | **Fetches Data:** Gathers the latest data from Google Analytics (users, sessions, conversions)     |
| Get Ad Platform Data         | HTTP Request        | Retrieves advertising platform data            | Workflow Configuration     | Calculate KPIs (MAU, LTV, CAC) | **Fetches Data:** Gathers the latest data from your advertising platform (ad spend)                |
| Get CRM Data                | HTTP Request        | Retrieves CRM data (customers, revenue)        | Workflow Configuration     | Calculate KPIs (MAU, LTV, CAC) | **Fetches Data:** Gathers the latest data from your CRM (new customers, revenue)                   |
| Calculate KPIs (MAU, LTV, CAC) | Code               | Calculates business KPIs from fetched data     | Get Google Analytics Data, Get Ad Platform Data, Get CRM Data | Write Data to Google Sheets          | **Calculates KPIs:** Processes the raw data to calculate essential business metrics                |
| Write Data to Google Sheets  | Google Sheets       | Logs KPIs into Google Sheet for historical data | Calculate KPIs (MAU, LTV, CAC) | Create Google Slides Report    | **Logs Historical Data:** Appends the newly calculated KPIs to a Google Sheet                      |
| Create Google Slides Report  | Google Slides       | Generates the Google Slides report presentation | Write Data to Google Sheets | Check for Errors               | **Generates a Report:** Creates a new Google Slides presentation to serve as the main report       |
| Check for Errors             | If                  | Checks if error occurred and routes flow       | Create Google Slides Report | Send Report Email, Send Error Notification | **Provides Error Alerts:** Sends notification to Slack on failure                                 |
| Send Report Email            | Gmail               | Sends the KPI summary email with report link   | Check for Errors (true)     | None                          | **Distributes the Report:** Emails a summary of the key metrics with link to full Google Slides report |
| Send Error Notification      | Slack               | Sends error notification to Slack channel      | Check for Errors (false)    | None                          | **Provides Error Alerts:** If any step fails, sends immediate notification to Slack channel        |
| Sticky Note                 | Sticky Note         | Documentation notes                            | None                       | None                          | See detailed notes below                                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Configure two intervals:  
     - Weekly: trigger every Monday at 09:00  
     - Monthly: trigger on the 1st day at 09:00  
   - No credentials needed.

2. **Create a Set node named "Workflow Configuration"**  
   - Assign the following variables:  
     - `gaPropertyId` (string): Google Analytics Property ID  
     - `adPlatformApiUrl` (string): API endpoint URL for ad platform  
     - `crmApiUrl` (string): API endpoint URL for CRM  
     - `reportSpreadsheetId` (string): Google Sheets ID for KPI storage  
     - `slidesTemplateId` (string): Google Slides template ID  
     - `reportRecipients` (string): Comma-separated emails  
     - `slackChannel` (string): Slack channel ID for errors  
     - `reportPeriodDays` (number): Days to look back (default 7)  
   - Connect Schedule Trigger output to this node.

3. **Create the Google Analytics node "Get Google Analytics Data"**  
   - Credential: Google Analytics OAuth2  
   - Property ID: Set value from `Workflow Configuration.gaPropertyId` (expression)  
   - Date Range: Custom, start date = now minus `reportPeriodDays`, end date = yesterday (use expressions)  
   - Metrics: active1DayUsers, sessions, screenPageViews, conversions  
   - Dimensions: leave empty or default  
   - Connect output of "Workflow Configuration" to this node.

4. **Create HTTP Request node "Get Ad Platform Data"**  
   - Credential: Predefined HTTP credential with ad platform access  
   - URL: from `Workflow Configuration.adPlatformApiUrl` (expression)  
   - Query parameters: `start_date` (now minus `reportPeriodDays`), `end_date` (yesterday)  
   - Connect output of "Workflow Configuration" to this node.

5. **Create HTTP Request node "Get CRM Data"**  
   - Credential: Predefined HTTP credential with CRM access  
   - URL: from `Workflow Configuration.crmApiUrl` (expression)  
   - Query parameters: `start_date` (now minus `reportPeriodDays`), `end_date` (yesterday)  
   - Connect output of "Workflow Configuration" to this node.

6. **Create Code node "Calculate KPIs (MAU, LTV, CAC)"**  
   - Inputs: Connect from all three data sources: Google Analytics, Ad Platform, CRM  
   - Code logic:  
     - Extract relevant fields: activeUsers, totalSpend, newCustomers, totalRevenue, totalCustomers, sessions, conversions  
     - Compute MAU, CAC, LTV, LTV:CAC ratio, conversion rate (with safe defaults)  
     - Return JSON object with KPIs and reporting dates  
   - Use expressions to access configuration node for dates.

7. **Create Google Sheets node "Write Data to Google Sheets"**  
   - Credential: Google Sheets OAuth2  
   - Operation: Append or update  
   - Sheet Name: "KPI Data"  
   - Document ID: from `Workflow Configuration.reportSpreadsheetId` (expression)  
   - Map columns automatically from incoming KPI JSON  
   - Connect output from KPI calculation node.

8. **Create Google Slides node "Create Google Slides Report"**  
   - Credential: Google Slides OAuth2  
   - Title: Set dynamically to "Analytics Report - YYYY-MM-DD" using expressions for current date  
   - Optional: Use template ID from `Workflow Configuration.slidesTemplateId` if supported by node  
   - Connect output of Google Sheets node.

9. **Create If node "Check for Errors"**  
   - Input: From Google Slides node  
   - Condition: Check if `error` key exists in JSON (operator: not exists)  
   - True output: to Send Report Email node  
   - False output: to Send Error Notification node

10. **Create Gmail node "Send Report Email"**  
    - Credential: Gmail OAuth2  
    - To: from `Workflow Configuration.reportRecipients` (expression)  
    - Subject: "Weekly/Monthly Analytics Report - YYYY-MM-DD" (expression)  
    - Message: HTML formatted summary of KPIs with period and link to Google Slides report (use expression to get presentation ID from Google Slides node)  
    - Attachments: none mandatory, but binary input enabled  
    - Connect from If node true output.

11. **Create Slack node "Send Error Notification"**  
    - Credential: Slack OAuth2  
    - Channel: from `Workflow Configuration.slackChannel` (expression)  
    - Message: Include error message from JSON or "Unknown error occurred" and current timestamp  
    - Connect from If node false output.

12. **Connect nodes respecting the data flow and dependency order:**  
    - Schedule Trigger → Workflow Configuration  
    - Workflow Configuration → all three data retrieval nodes  
    - All data retrieval nodes → Calculate KPIs  
    - Calculate KPIs → Write Data to Google Sheets  
    - Write Data to Google Sheets → Create Google Slides Report  
    - Create Google Slides Report → Check for Errors  
    - Check for Errors true → Send Report Email  
    - Check for Errors false → Send Error Notification

13. **Set up credentials** for all external services: Google Analytics, Google Sheets, Google Slides, Gmail (OAuth2), Slack (OAuth2), HTTP Request nodes (ad platform and CRM).

14. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| This workflow automates consistent marketing report generation and distribution, saving manual effort and ensuring timely updates. It is ideal for marketing teams, analysts, and business owners.                                                                                                                                                                                                                                                                                                                          | Workflow description and use case                                                                                           |
| To customize the reporting period, adjust the `reportPeriodDays` parameter in the "Workflow Configuration" node. For example, use 30 for monthly reports.                                                                                                                                                                                                                                                                                                                                                                 | Configuration customization                                                                                                 |
| The workflow includes error alerting via Slack to promptly notify responsible parties of failures, aiding quick incident response.                                                                                                                                                                                                                                                                                                                                                                                      | Error handling best practice                                                                                                |
| The "Create Google Slides Report" node currently creates a new presentation but can be enhanced to populate charts and KPI data dynamically using Google Slides API features.                                                                                                                                                                                                                                                                                                                                            | Potential workflow enhancement                                                                                             |
| Credential setup is critical: ensure OAuth2 credentials for Google services are correctly configured with necessary scopes, and HTTP requests are authenticated using appropriate tokens or API keys.                                                                                                                                                                                                                                                                                                                     | Credential management                                                                                                       |
| For detailed instructions on n8n Google Analytics node usage and Google Sheets integration, refer to official n8n documentation: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googleAnalytics/ and https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.googleSheets/                                                                                                                                                                                                                       | Official n8n documentation                                                                                                  |
| Slack notification messages use a simple templated text format; for richer formatting or attachments, Slack Block Kit can be implemented in future iterations.                                                                                                                                                                                                                                                                                                                                                           | Slack message formatting                                                                                                   |

---

**Disclaimer:** The provided description and workflow analysis originate exclusively from an automated n8n workflow. All content respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.