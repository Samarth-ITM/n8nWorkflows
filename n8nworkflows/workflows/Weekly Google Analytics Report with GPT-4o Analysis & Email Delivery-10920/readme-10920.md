Weekly Google Analytics Report with GPT-4o Analysis & Email Delivery

https://n8nworkflows.xyz/workflows/weekly-google-analytics-report-with-gpt-4o-analysis---email-delivery-10920


# Weekly Google Analytics Report with GPT-4o Analysis & Email Delivery

### 1. Workflow Overview

This workflow automates the process of generating a **weekly Google Analytics performance report**, enhanced with **AI-driven analysis**, and delivers the report via email. It targets digital marketing teams or managers who require timely insights into user engagement and page views trends.

The workflow is logically divided into the following functional blocks:

- **1.1 Scheduled Trigger & Date Range Generation:** Automates weekly execution and calculates relevant date ranges for the current and previous weeks.
- **1.2 Data Retrieval from Google Analytics:** Fetches user and page view metrics for the two defined weekly periods.
- **1.3 Data Extraction and Comparison:** Aggregates raw GA data and calculates week-over-week percentage changes.
- **1.4 AI-Based Report Generation:** Uses OpenAI GPT-4o-mini to analyze the data and produce a structured performance summary.
- **1.5 Chart Creation & HTML Report Composition:** Generates a visual comparison chart and composes an HTML report embedding both text and graphics.
- **1.6 PDF Conversion & Email Delivery:** Converts the report to PDF and sends it via Gmail to the designated recipient.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger & Date Range Generation

- **Overview:** Initiates the workflow on a weekly schedule and computes the exact date ranges for the current and previous week to query Google Analytics.
- **Nodes Involved:** `Schedule Trigger`, `Generate Date Range`
- **Node Details:**
  - **Schedule Trigger**
    - Type: Trigger node that fires periodically.
    - Configuration: Set to trigger every week.
    - Outputs: Activates downstream nodes to start the data processing.
    - Failures: Rare, but possible if n8n instance is down or scheduler misconfigured.
  - **Generate Date Range**
    - Type: Code node (JavaScript).
    - Configuration: Calculates ISO date strings:
      - This week: last 7 days (startThisWeek to endThisWeek).
      - Last week: 7–14 days ago (startLastWeek to endLastWeek).
    - Key variables output: `startThisWeek`, `endThisWeek`, `startLastWeek`, `endLastWeek`.
    - Inputs: Trigger output.
    - Outputs: Provides parameters for GA data retrieval.
    - Edge cases: Timezone issues or date miscalculations if system clock skewed.

#### 1.2 Data Retrieval from Google Analytics

- **Overview:** Queries Google Analytics API for user and page view metrics for both weekly periods.
- **Nodes Involved:** `GA This Week`, `GA Last Week`
- **Node Details:**
  - Both nodes are Google Analytics nodes configured similarly:
    - Type: `googleAnalytics` node, version 2.
    - Credentials: Connected to a Google Analytics OAuth2 account.
    - Property ID: 456857722 (BinaryTech).
    - Metrics requested: `totalUsers`, `screenPageViews`.
    - Dimensions: none (empty).
    - Date range: uses dynamic dates from `Generate Date Range` output.
    - Output: Raw GA data per period.
    - Possible failures: Auth token expiry, API quota exceeded, invalid property ID, network errors.

#### 1.3 Data Extraction and Comparison

- **Overview:** Processes raw GA data to sum metrics and calculate percentage differences between the two weeks.
- **Nodes Involved:** `Extract GA This Week`, `Extract GA Last Week`, `Merge`, `Calculate Differences`
- **Node Details:**
  - **Extract GA This Week / Extract GA Last Week**
    - Type: Code nodes (JavaScript).
    - Function: Aggregate metrics (`totalUsers`, `screenPageViews`) across all input items for the respective week.
    - Output: JSON object with summed values under keys `this_week` or `last_week`.
    - Inputs: GA nodes’ outputs.
    - Edge cases: Empty data sets, non-numeric values.
  - **Merge**
    - Type: Merge node (combine by position).
    - Function: Combines the two aggregated outputs into a single array for comparison.
    - Inputs: Outputs from both Extract nodes.
    - Outputs: Combined data array.
  - **Calculate Differences**
    - Type: Code node.
    - Function: Takes the combined data, calculates percentage changes for users and page views.
    - Function `pctChange` handles divide-by-zero cases by returning 100%.
    - Outputs detailed JSON with current and previous week metrics plus % change.
    - Edge cases: Zero previous values, which lead to 100% change indicators.

#### 1.4 AI-Based Report Generation

- **Overview:** Uses GPT-4o-mini to analyze the metrics and generate a descriptive HTML summary report with actionable insights.
- **Nodes Involved:** `Message a model1`
- **Node Details:**
  - Type: OpenAI node.
  - Configuration:
    - Model: GPT-4o-mini.
    - Prompt includes:
      - Instructions to create a concise summary with bullet points.
      - Identify the most and least affected metrics.
      - Assess severity of drops.
      - Provide two actionable recommendations.
      - Input data is stringified JSON of calculated differences.
    - Output: HTML-formatted report only (no extraneous text).
  - Inputs: Data from `Calculate Differences`.
  - Possible failures: API limits, prompt errors, malformed input data.

#### 1.5 Chart Creation & HTML Report Composition

- **Overview:** Constructs a bar chart comparing week metrics, creates an HTML report embedding text and chart.
- **Nodes Involved:** `Generate chart`, `Report template`
- **Node Details:**
  - **Generate chart**
    - Type: Code node.
    - Function:
      - Creates a QuickChart-compatible JSON config for a bar chart with two datasets (This Week, Last Week).
      - Encodes chart config URL for embedding.
      - Returns the AI-generated report text and chart URL.
    - Inputs: Output from `Message a model1`.
    - Edge cases: Chart URL encoding errors, missing data.
  - **Report template**
    - Type: HTML node.
    - Function:
      - Builds full HTML page embedding the AI report and the chart image.
      - Uses simple inline CSS for styling.
    - Inputs: Output from `Generate chart`.
    - Outputs: HTML content ready for PDF conversion.

#### 1.6 PDF Conversion & Email Delivery

- **Overview:** Converts HTML report to PDF and emails it to the manager.
- **Nodes Involved:** `HTML to PDF`, `Send a message`
- **Node Details:**
  - **HTML to PDF**
    - Type: HTML to PDF node.
    - Credentials: Uses an external HTML-to-PDF API service.
    - Inputs: HTML content from `Report template`.
    - Outputs: PDF file URL.
    - Possible failures: API key issues, conversion errors, service downtime.
  - **Send a message**
    - Type: Gmail node.
    - Credentials: Gmail OAuth2 credentials.
    - Configuration:
      - Sends an HTML email to `personal@example.com` (replace with actual).
      - Email body embeds the report HTML and a link to the PDF.
      - Subject: “GA: Weekly report”.
    - Inputs: PDF URL from `HTML to PDF`, report HTML from `Generate chart`.
    - Edge cases: Gmail quota, OAuth token expiry, email formatting issues.

---

### 3. Summary Table

| Node Name          | Node Type                              | Functional Role                             | Input Node(s)            | Output Node(s)          | Sticky Note                                                                                                                                |
|--------------------|--------------------------------------|---------------------------------------------|--------------------------|-------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger    | scheduleTrigger                      | Weekly workflow trigger                      |                          | Generate Date Range      |                                                                                                                                             |
| Generate Date Range | code                                | Calculate date ranges for this and last week| Schedule Trigger          | GA This Week, GA Last Week | ## Get data from Google Analytics                                                                                                         |
| GA This Week       | googleAnalytics (v2)                  | Fetch GA data for current week               | Generate Date Range       | Extract GA This Week     | ## Get data from Google Analytics                                                                                                         |
| GA Last Week       | googleAnalytics (v2)                  | Fetch GA data for previous week              | Generate Date Range       | Extract GA Last Week     | ## Get data from Google Analytics                                                                                                         |
| Extract GA This Week| code                                | Aggregate this week’s GA metrics             | GA This Week              | Merge                   | ## Compare the data from the two weeks and calculate the percentage difference.                                                            |
| Extract GA Last Week| code                                | Aggregate last week’s GA metrics             | GA Last Week              | Merge                   | ## Compare the data from the two weeks and calculate the percentage difference.                                                            |
| Merge              | merge (combine by position)          | Combine aggregated weekly metrics            | Extract GA This Week, Extract GA Last Week | Calculate Differences | ## Compare the data from the two weeks and calculate the percentage difference.                                                            |
| Calculate Differences| code                               | Compute percentage changes between weeks     | Merge                    | Message a model1         | ## Compare the data from the two weeks and calculate the percentage difference.                                                            |
| Message a model1    | openAi (GPT-4o-mini)                 | AI analysis & report generation               | Calculate Differences     | Generate chart           | ## Generate Report using AI                                                                                                                |
| Generate chart      | code                                | Build chart URL & prepare report data        | Message a model1          | Report template          | ## Generate Chart                                                                                                                          |
| Report template     | html                                | Compose final HTML report                      | Generate chart            | HTML to PDF              | ## SGenerate PDF from Report                                                                                                               |
| HTML to PDF        | htmlcsstopdf                        | Convert HTML report to PDF                     | Report template           | Send a message           | ## SGenerate PDF from Report                                                                                                               |
| Send a message      | gmail                              | Email the PDF report to the manager           | HTML to PDF               |                         | ## Send report to the person in charge                                                                                                    |
| Sticky Note        | stickyNote                         | Overview & setup instructions                  |                          |                         | ## Overview (Detailed explanation and setup links)                                                                                        |
| Sticky Note5       | stickyNote                         | Label for GA data retrieval block              |                          |                         | ## Get data from Google Analytics                                                                                                         |
| Sticky Note6       | stickyNote                         | Label for data comparison block                 |                          |                         | ## Compare the data from the two weeks and calculate the percentage difference.                                                            |
| Sticky Note7       | stickyNote                         | Label for AI report generation block            |                          |                         | ## Generate Report using AI                                                                                                                |
| Sticky Note8       | stickyNote                         | Label for email sending block                    |                          |                         | ## Send report to the person in charge                                                                                                    |
| Sticky Note9       | stickyNote                         | Label for chart generation block                 |                          |                         | ## Generate Chart                                                                                                                          |
| Sticky Note10      | stickyNote                         | Label for PDF generation block                   |                          |                         | ## SGenerate PDF from Report                                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Scheduled Trigger Node**
   - Type: `Schedule Trigger`
   - Configuration: Set interval to `1 week`.
   - Purpose: To start the workflow weekly.

2. **Add a Code Node “Generate Date Range”**
   - Type: `Code`
   - JavaScript code to calculate `startThisWeek`, `endThisWeek`, `startLastWeek`, `endLastWeek` as ISO date strings:
     ```js
     const today = new Date();
     const endThisWeek = today.toISOString().split("T")[0];
     const startThisWeekDate = new Date();
     startThisWeekDate.setDate(startThisWeekDate.getDate() - 7);
     const startThisWeek = startThisWeekDate.toISOString().split("T")[0];
     const endLastWeekDate = new Date();
     endLastWeekDate.setDate(endLastWeekDate.getDate() - 7);
     const endLastWeek = endLastWeekDate.toISOString().split("T")[0];
     const startLastWeekDate = new Date();
     startLastWeekDate.setDate(startLastWeekDate.getDate() - 14);
     const startLastWeek = startLastWeekDate.toISOString().split("T")[0];
     return [{ json: { startThisWeek, endThisWeek, startLastWeek, endLastWeek } }];
     ```
   - Connect output of schedule trigger to this node.

3. **Add Two Google Analytics Nodes: “GA This Week” and “GA Last Week”**
   - Type: `Google Analytics`
   - Credentials: Configure with Google Analytics OAuth2 credentials.
   - Property ID: Set to the GA property (e.g., `456857722`).
   - Metrics: Request `totalUsers` and `screenPageViews`.
   - Date Range:
     - For “GA This Week”: `startDate` = `{{$json.startThisWeek}}`, `endDate` = `{{$json.endThisWeek}}`
     - For “GA Last Week”: `startDate` = `{{$json.startLastWeek}}`, `endDate` = `{{$json.endLastWeek}}`
   - Dimensions: None.
   - Connect outputs of “Generate Date Range” to both GA nodes.

4. **Add Two Code Nodes for Data Extraction: “Extract GA This Week” and “Extract GA Last Week”**
   - Type: `Code`
   - Logic: Sum the `totalUsers` and `screenPageViews` values across all incoming items.
   - Example code (same for both, changing output key accordingly):
     ```js
     const data = $input.all().map(item => item.json);
     const result = data.reduce((acc, item) => {
       acc.totalUsers += Number(item.totalUsers) || 0;
       acc.screenPageViews += Number(item.screenPageViews) || 0;
       return acc;
     }, { totalUsers: 0, screenPageViews: 0 });
     return [{ json: { this_week: result } }]; // or { last_week: result } for last week node
     ```
   - Connect “GA This Week” to “Extract GA This Week”.
   - Connect “GA Last Week” to “Extract GA Last Week”.

5. **Add a Merge Node**
   - Type: `Merge`
   - Mode: Combine by position.
   - Connect output 0 from “Extract GA This Week” and output 0 from “Extract GA Last Week”.

6. **Add a Code Node “Calculate Differences”**
   - Type: `Code`
   - Code to compute percentage changes and prepare comparison JSON:
     ```js
     const thisWeek = $input.first().json.this_week;
     const lastWeek = $input.first().json.last_week;
     function pctChange(current, previous) {
       if (previous === 0) return 100;
       return ((current - previous) / previous) * 100;
     }
     return [{
       json: {
         users_this_week: thisWeek.totalUsers,
         users_last_week: lastWeek.totalUsers,
         users_change_pct: pctChange(thisWeek.totalUsers, lastWeek.totalUsers),
         pv_this_week: thisWeek.screenPageViews,
         pv_last_week: lastWeek.screenPageViews,
         pv_change_pct: pctChange(thisWeek.screenPageViews, lastWeek.screenPageViews),
       }
     }];
     ```
   - Connect “Merge” output to this node.

7. **Add OpenAI Node “Message a model1”**
   - Type: OpenAI (LangChain) node.
   - Credentials: Configure with OpenAI API key.
   - Model: Select `gpt-4o-mini`.
   - Message content: Provide prompt requesting summary, analysis, severity assessment, and recommendations, embedding the JSON data as input.
   - Ensure output is HTML only, per instructions.
   - Connect output from “Calculate Differences”.

8. **Add Code Node “Generate chart”**
   - Type: `Code`
   - Build QuickChart URL with bar chart config comparing current and previous week metrics.
   - Extract AI report text and prepare output JSON with `report` and `chart_url`.
   - Example code snippet:
     ```js
     const chartConfig = {
       type: "bar",
       data: {
         labels: ["PageView", "Users"],
         datasets: [
           {
             label: "This Week",
             data: [
               $('Calculate Differences').first().json.pv_this_week,
               $('Calculate Differences').first().json.users_this_week,
             ],
           },
           {
             label: "Last Week",
             data: [
               $('Calculate Differences').first().json.pv_last_week,
               $('Calculate Differences').first().json.users_last_week,
             ],
           },
         ],
       },
     };
     const encoded = encodeURIComponent(JSON.stringify(chartConfig));
     return {
       json: {
         report: $input.first().json.output[0].content[0].text,
         chart_url: `https://quickchart.io/chart?width=400&height=200&c=${encoded}`,
       },
     };
     ```
   - Connect output from “Message a model1”.

9. **Add HTML Node “Report template”**
   - Type: `HTML`
   - HTML content:
     ```html
     <html>
       <body style="font-family: Arial; padding: 20px">
         <div id="report">{{ $json.report }}</div>
         <div id="chart">
           <h3>Comparison Chart</h3>
           <img src={{ $json.chart_url }} />
         </div>
       </body>
     </html>
     ```
   - Connect output from “Generate chart”.

10. **Add HTML to PDF Node**
    - Type: `HTML to PDF`
    - Credentials: Configure with HTML to PDF API key.
    - Input: HTML content from “Report template”.
    - Connect output from “Report template”.

11. **Add Gmail Node “Send a message”**
    - Type: `Gmail` (OAuth2)
    - Credentials: Configure Gmail OAuth2 credentials.
    - Send To: Manager’s email (replace placeholder).
    - Subject: “GA: Weekly report”
    - Message (HTML):
      ```html
      <html>
        <body style="font-family: Arial; padding: 20px">
          <div id="report">
            {{ $('Generate chart').item.json.report }}
          </div>
          <div id="chart">
            <h3>Comparison Chart</h3>
            Click to view the <a href="{{ $json.pdf_url }}">PDF Report</a>
          </div>
        </body>
      </html>
      ```
    - Connect output from “HTML to PDF”.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow automatically fetches GA data for the two most recent weeks, compares and calculates variances, uses AI (GPT-4o-mini) for analysis and report generation, creates charts, exports PDF, and sends email.                                                                                                  | Overall workflow summary                                                                                     |
| Google Cloud setup required: Create credentials and enable Gmail API, Google Analytics Admin API, and Google Analytics Data API.                                                                                                                                                                             | https://console.cloud.google.com/apis/credentials <br> https://console.cloud.google.com/apis/dashboard       |
| HTML to PDF conversion uses external API (e.g., pdfmunk.com) requiring API key configuration.                                                                                                                                                                                                                 | https://pdfmunk.com/api-keys                                                                                 |
| Gmail OAuth2 credentials should be configured with proper scopes to allow sending emails.                                                                                                                                                                                                                      | Gmail API OAuth2 setup                                                                                        |
| Chart generation uses QuickChart.io service by encoding chart config into a URL to embed bar charts dynamically.                                                                                                                                                                                             | https://quickchart.io/                                                                                        |
| AI prompt enforces HTML-only output with structured tags (`<h4>`, `<ul>`, `<li>`, `<table>`) for consistent report formatting.                                                                                                                                                                              | Prompt design for OpenAI node                                                                                 |

---

This documentation fully describes the workflow structure, node configurations, logical flow, and setup instructions to enable advanced users or AI agents to understand, replicate, and troubleshoot the Weekly Google Analytics Report workflow.