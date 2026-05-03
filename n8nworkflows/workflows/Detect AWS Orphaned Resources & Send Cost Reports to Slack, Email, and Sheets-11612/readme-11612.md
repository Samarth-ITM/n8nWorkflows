Detect AWS Orphaned Resources & Send Cost Reports to Slack, Email, and Sheets

https://n8nworkflows.xyz/workflows/detect-aws-orphaned-resources---send-cost-reports-to-slack--email--and-sheets-11612


# Detect AWS Orphaned Resources & Send Cost Reports to Slack, Email, and Sheets

---

## 1. Workflow Overview

This workflow automates the detection of orphaned AWS resources across multiple regions and generates comprehensive cost reports sent to Slack, email, and Google Sheets. It targets FinOps and Cloud Operations teams aiming to optimize cloud spend by identifying unattached EBS volumes, old snapshots (older than 90 days), and unassociated Elastic IPs. The workflow is structured into logical blocks that handle configuration, scanning AWS resources via Lambda functions, processing and aggregating data, generating reports, and dispatching alerts through multiple channels.

### Logical Blocks

- **1.1 Initialization & Scheduling:** Configure scan parameters and trigger the scan on a weekly schedule.
- **1.2 AWS Resource Scanning:** Invoke AWS Lambda functions to scan for orphaned EBS volumes, snapshots, and Elastic IPs in specified regions.
- **1.3 Data Processing:** Parse Lambda responses, calculate costs, check compliance (tagging), and prepare detailed resource data.
- **1.4 Data Aggregation & Summarization:** Combine all resource data, calculate total and breakdown statistics, rank expensive resources, and assess compliance.
- **1.5 Reporting & Alerts:** Generate HTML and CSV reports, send Slack alerts, email detailed reports, and log results in Google Sheets.
- **1.6 Conditional Branching:** Handle scenarios when orphaned resources are found versus no waste detected, adjusting notifications accordingly.
- **1.7 Documentation & Guidance:** Sticky notes provide prerequisites, configuration instructions, and troubleshooting tips.

---

## 2. Block-by-Block Analysis

### 2.1 Initialization & Scheduling

- **Overview:** Sets up global configuration parameters, assigns the scanning region and timestamp, and triggers the workflow on a weekly schedule.
- **Nodes Involved:**  
  - Weekly Scan Trigger1  
  - Initialize Config  
  - Set Region Variables  
  - Workflow Overview (Sticky Note)  
  - Sticky Note (Prerequisites)  
  - Sticky Note2 (Configuration Instructions)

- **Node Details:**

  - **Weekly Scan Trigger1**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow every Monday at 8:00 UTC.  
    - Config: Cron expression `"0 8 * * 1"` for weekly execution.  
    - Input: None  
    - Output: Triggers `Initialize Config`.  
    - Failure Modes: Cron misconfiguration, time zone mismatch.

  - **Initialize Config**  
    - Type: Set  
    - Role: Defines configuration parameters including AWS regions to scan, Slack channel, email recipients, required tags, cost assumptions, and cleanup enablement.  
    - Config: JSON with keys like `awsRegions`, `slackChannel`, `emailRecipients`, `requiredTags`, `snapshotAgeDays`, etc.  
    - Input: Trigger from Schedule  
    - Output: To `Set Region Variables`.  
    - Failure Modes: Misconfigured parameters, invalid JSON formatting.

  - **Set Region Variables**  
    - Type: Set  
    - Role: Assigns the current AWS region for scanning (first in list) and the scan start time (ISO format).  
    - Config: Uses expressions to get the first region and current timestamp.  
    - Input: From Initialize Config  
    - Output: Triggers AWS Lambda scan nodes.  
    - Failure Modes: Empty `awsRegions` array causing expression failure.

  - **Sticky Notes** provide contextual guidance on prerequisites, configuration, and workflow overview.

---

### 2.2 AWS Resource Scanning

- **Overview:** Executes AWS Lambda functions to scan for orphaned resources per region and resource type.
- **Nodes Involved:**  
  - Scan unattached EBS Volumes  
  - Scan EBS Snapshots  
  - Scan Elastic IPs  
  - Sticky Note1 (Lambda Scans Explanation)

- **Node Details:**

  - **Scan unattached EBS Volumes**  
    - Type: AWS Lambda  
    - Role: Calls Lambda function named `aws-orphaned-resource-scanner` with `resourceType` set to `volumes` and current region.  
    - Config: Payload includes dynamic region assignment from `currentRegion`.  
    - Credentials: AWS IAM with EC2 read + Lambda invoke permissions.  
    - Input: From `Set Region Variables`  
    - Output: To `Process EBS Volumes`.  
    - Failure Modes: Lambda invoke errors (403), AWS permissions issues, timeout.

  - **Scan EBS Snapshots**  
    - Same as above but with `resourceType` set to `snapshots`.  
    - Output: To `Process Snapshots`.

  - **Scan Elastic IPs**  
    - Same as above but with `resourceType` set to `addresses`.  
    - Output: To `Process Elastic IPs`.

  - **Sticky Note1** explains the Lambda scanning step and links to the GitHub repo for the Lambda function.

---

### 2.3 Data Processing

- **Overview:** Parses the JSON response from each Lambda scan, calculates monthly and annual costs based on size or count, checks for missing required tags, and enriches resource data with compliance information.
- **Nodes Involved:**  
  - Process EBS Volumes  
  - Process Snapshots  
  - Process Elastic IPs  
  - Sticky Note4 (Processing Explanation)

- **Node Details:**

  - **Process EBS Volumes**  
    - Type: Code  
    - Role: Parses volumes data, calculates costs ($0.10 per GB-month), identifies missing tags from required tags list, and prepares enhanced resource objects.  
    - Key Expressions: Uses fixed cost per GB, iterates volumes to sum size and detect tags.  
    - Input: Lambda scan result for volumes  
    - Output: To `Aggregate All Resources`.  
    - Edge Cases: No volumes found, malformed tag data, zero-size volumes.

  - **Process Snapshots**  
    - Type: Code  
    - Role: Parses snapshot data, calculates cost ($0.05 per GB-month), computes age in days, checks missing tags, outputs enhanced snapshot info.  
    - Input: Lambda scan result for snapshots  
    - Output: To `Aggregate All Resources`.

  - **Process Elastic IPs**  
    - Type: Code  
    - Role: Counts unassociated Elastic IPs, calculates cost ($3.60 per month per IP), checks tags, enriches resource list.  
    - Input: Lambda scan result for addresses  
    - Output: To `Aggregate All Resources`.

  - **Sticky Note4** clarifies the processing logic and cost calculations.

---

### 2.4 Data Aggregation & Summarization

- **Overview:** Combines all processed resource types into a single dataset, calculates totals and breakdown statistics by type and region, ranks the top 5 most expensive orphaned resources, and assesses compliance metrics.
- **Nodes Involved:**  
  - Aggregate All Resources  
  - Calculate Summary Stats  
  - If Resources Found  
  - Sticky Note5 (Combination and Summary)

- **Node Details:**

  - **Aggregate All Resources**  
    - Type: Aggregate  
    - Role: Merges all resource arrays from volumes, snapshots, and IPs into one field `data`.  
    - Input: Processed resource nodes (volumes, snapshots, IPs)  
    - Output: To `Calculate Summary Stats`.

  - **Calculate Summary Stats**  
    - Type: Code  
    - Role:  
      - Iterates aggregated data to compute:  
        - Total monthly and annual cost  
        - Resource counts by type and region  
        - Number of resources with missing tags  
        - Compliance rate (% of resources properly tagged)  
        - Top 5 expensive resources ranked by monthly cost  
      - Returns structured JSON for downstream reporting nodes.  
    - Input: Aggregated resource data  
    - Output: To `If Resources Found`.

  - **If Resources Found**  
    - Type: If (Boolean Condition)  
    - Role: Checks if total orphaned resources found > 0 to branch alerting logic.  
    - Condition: `totalResourcesFound > 0`  
    - Output:  
      - True: Proceed to generate reports and alerts for resources found.  
      - False: Send "no resources found" alerts and logs.

  - **Sticky Note5** explains this combination and summarization step.

---

### 2.5 Reporting & Alerts

- **Overview:** Generates detailed HTML and CSV reports, dispatches Slack alerts and emails, and logs results to Google Sheets. Different flows occur based on whether orphaned resources are detected.
- **Nodes Involved:**  
  - Generate HTML Report  
  - Generate CSV Export  
  - Send Slack Alert  
  - Send Email Report  
  - Log to Google Sheets (Found)  
  - None Found Slack Message  
  - Clean Scan CSV Export  
  - Append or update row in sheet  
  - Sticky Note6 (Alerts with Resources Found)  
  - Sticky Note7 (Alerts with No Resources Found)  
  - Sticky Note3 (Common Issues)

- **Node Details:**

  - **Generate HTML Report**  
    - Type: Code  
    - Role: Creates a professional HTML report summarizing total costs, compliance, top 5 expensive resources, breakdown by resource type and region, with styled tables and color-coded compliance indicators.  
    - Input: Summary stats from `If Resources Found` (true branch)  
    - Output: To `Send Email Report`.

  - **Generate CSV Export**  
    - Type: Code  
    - Role: Converts detailed resource data into CSV format with columns like Resource Type, Region, Resource ID, Costs, Risk Score, Compliance, Missing Tags, Age, Create Date. Also prepares structured data for Google Sheets upload.  
    - Input: Summary stats and aggregated full data  
    - Output: To `Log to Google Sheets (Found)`.

  - **Send Slack Alert**  
    - Type: Slack  
    - Role: Sends immediate alert to configured Slack channel with summary stats, compliance, and top offenders. Includes dynamic emojis based on compliance rate.  
    - Input: Summary stats  
    - Output: None  
    - Failure Modes: Slack API auth errors, invalid channel.

  - **Send Email Report**  
    - Type: Gmail  
    - Role: Sends the generated HTML report by email to configured recipients with subject line including annual waste estimate.  
    - Credentials: Gmail OAuth2  
    - Input: HTML report from `Generate HTML Report`  
    - Output: None  
    - Failure Modes: Gmail OAuth errors, spam filtering.

  - **Log to Google Sheets (Found)**  
    - Type: Google Sheets  
    - Role: Appends or updates a row in a configured sheet with detailed scan summary data and resource counts.  
    - Credentials: Google Sheets OAuth2  
    - Input: CSV export node data  
    - Output: None  
    - Failure Modes: Sheet name mismatch, OAuth token expiry.

  - **None Found Slack Message**  
    - Type: Slack  
    - Role: Sends a confirmation message indicating no orphaned resources found, with scan date and region info.  
    - Input: From `If Resources Found` (false branch)  
    - Output: None.

  - **Clean Scan CSV Export**  
    - Type: Code  
    - Role: Creates a minimal CSV log entry indicating a clean scan (no waste), with compliance at 100%, zero resource counts, and notes.  
    - Input: Summary stats from false branch  
    - Output: To `Append or update row in sheet`.

  - **Append or update row in sheet**  
    - Type: Google Sheets  
    - Role: Logs the clean scan summary to the same Google Sheets audit trail as the found resources flow.  
    - Credentials: Google Sheets OAuth2.

  - **Sticky Note6 and Sticky Note7** describe the alerting logic for both scenarios.

  - **Sticky Note3** provides common troubleshooting information for Lambda 403 errors, Gmail OAuth issues, and Google Sheets syncing.

---

## 3. Summary Table

| Node Name                  | Node Type          | Functional Role                                      | Input Node(s)                       | Output Node(s)                                             | Sticky Note                                                                                                   |
|----------------------------|--------------------|-----------------------------------------------------|-----------------------------------|------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| Workflow Overview          | Sticky Note        | Overview and usage instructions                      | None                              | None                                                       | See workflow overview and quick start instructions                                                          |
| Weekly Scan Trigger1       | Schedule Trigger   | Triggers workflow weekly on Monday 8:00 UTC         | None                              | Initialize Config                                           |                                                                                                              |
| Initialize Config          | Set                | Sets configuration parameters                        | Weekly Scan Trigger1              | Set Region Variables                                       |                                                                                                              |
| Set Region Variables       | Set                | Assigns current region and scan start time           | Initialize Config                 | Scan unattached EBS Volumes, Scan EBS Snapshots, Scan Elastic IPs |                                                                                                              |
| Scan unattached EBS Volumes| AWS Lambda          | Scans EBS volumes in region                          | Set Region Variables              | Process EBS Volumes                                        | Lambda scan explanation with link to GitHub Lambda function                                                  |
| Scan EBS Snapshots         | AWS Lambda          | Scans snapshots in region                            | Set Region Variables              | Process Snapshots                                         |                                                                                                              |
| Scan Elastic IPs           | AWS Lambda          | Scans unassociated Elastic IPs                       | Set Region Variables              | Process Elastic IPs                                       |                                                                                                              |
| Process EBS Volumes        | Code                | Parses volume data, calculates cost & tags          | Scan unattached EBS Volumes       | Aggregate All Resources                                   |                                                                                                              |
| Process Snapshots          | Code                | Parses snapshot data, calculates cost & age          | Scan EBS Snapshots                | Aggregate All Resources                                   |                                                                                                              |
| Process Elastic IPs        | Code                | Parses Elastic IP data, calculates cost & tags       | Scan Elastic IPs                  | Aggregate All Resources                                   |                                                                                                              |
| Aggregate All Resources    | Aggregate           | Combines all resource data into one array            | Process EBS Volumes, Process Snapshots, Process Elastic IPs | Calculate Summary Stats                                   |                                                                                                              |
| Calculate Summary Stats    | Code                | Calculates totals, breakdowns, compliance, top 5     | Aggregate All Resources           | If Resources Found                                       |                                                                                                              |
| If Resources Found         | If                  | Branches based on whether orphaned resources exist  | Calculate Summary Stats           | Generate HTML Report (T), Send Slack Alert (T), Generate CSV Export (T), None Found Slack Message (F), Clean Scan CSV Export (F) |                                                                                                              |
| Generate HTML Report       | Code                | Creates detailed HTML report                          | If Resources Found (true)         | Send Email Report                                        |                                                                                                              |
| Generate CSV Export        | Code                | Creates CSV data and structured array for sheets     | If Resources Found (true)         | Log to Google Sheets (Found)                             |                                                                                                              |
| Send Slack Alert           | Slack               | Sends alert message to Slack channel                  | If Resources Found (true)         | None                                                    |                                                                                                              |
| Send Email Report          | Gmail               | Sends HTML report email                               | Generate HTML Report              | None                                                    |                                                                                                              |
| Log to Google Sheets (Found)| Google Sheets      | Logs detailed scan info                               | Generate CSV Export               | None                                                    |                                                                                                              |
| None Found Slack Message   | Slack               | Sends "no waste found" confirmation                   | If Resources Found (false)        | None                                                    |                                                                                                              |
| Clean Scan CSV Export      | Code                | Creates CSV log for clean scan                        | If Resources Found (false)        | Append or update row in sheet                            |                                                                                                              |
| Append or update row in sheet | Google Sheets    | Logs clean scan info                                  | Clean Scan CSV Export             | None                                                    |                                                                                                              |
| Sticky Note                | Sticky Note         | Prerequisites checklist                               | None                              | None                                                    | Checklist on AWS IAM, Lambda, n8n credentials, setup steps                                                  |
| Sticky Note2               | Sticky Note         | Configuration instructions                            | None                              | None                                                    | Lists required tags, snapshot age, regions, schedule, alert channel                                        |
| Sticky Note3               | Sticky Note         | Common issues and troubleshooting tips                | None                              | None                                                    | Troubleshooting Lambda 403, Gmail OAuth, Google Sheets sync                                                |
| Sticky Note4               | Sticky Note         | Explains resource processing logic                    | None                              | None                                                    | Describes parsing and cost calculation from Lambda responses                                               |
| Sticky Note5               | Sticky Note         | Explains aggregation and summarization                | None                              | None                                                    | Describes combining all resource data and generating summary stats                                         |
| Sticky Note6               | Sticky Note         | Multi-channel alerts when resources found             | None                              | None                                                    | Explains parallel Slack, Email, Sheets alerts                                                             |
| Sticky Note7               | Sticky Note         | Multi-channel alerts when no resources found          | None                              | None                                                    | Explains Slack and Sheets logging for clean scans                                                         |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: `Weekly Scan Trigger1`  
   - Type: Schedule Trigger  
   - Cron Expression: `0 8 * * 1` (Every Monday at 08:00 UTC)  
   - No credentials needed.

2. **Create Set Node for Configuration**  
   - Name: `Initialize Config`  
   - Type: Set  
   - Mode: Raw JSON  
   - JSON Content:  
     ```json
     {
       "awsRegions": ["us-east-1", "us-west-2"],
       "slackChannel": "#cloud-ops",
       "emailRecipients": "finops@company.com",
       "requiredTags": ["Environment", "Owner", "CostCenter"],
       "snapshotAgeDays": 90,
       "stoppedInstanceDays": 30,
       "ebsCostPerGbMonth": 0.1,
       "snapshotCostPerGbMonth": 0.05,
       "elasticIpCostPerMonth": 3.6,
       "enableCleanup": false
     }
     ```  
   - Connect: Output from `Weekly Scan Trigger1`.

3. **Create Set Node for Region Variables**  
   - Name: `Set Region Variables`  
   - Type: Set  
   - Assignments:  
     - `currentRegion` = `={{ $json.awsRegions[0] }}`  
     - `scanStartTime` = `={{ $now.toISO() }}`  
   - Connect: Output from `Initialize Config`.

4. **Create Three AWS Lambda Nodes for Scanning**  
   - Names:  
     - `Scan unattached EBS Volumes`  
     - `Scan EBS Snapshots`  
     - `Scan Elastic IPs`  
   - Type: AWS Lambda  
   - Credentials: AWS IAM user with EC2 read and Lambda invoke permissions.  
   - Parameters for each:  
     - `function`: `aws-orphaned-resource-scanner`  
     - `payload`: JSON with keys `region` (from `currentRegion`) and `resourceType` set respectively to `volumes`, `snapshots`, and `addresses`.  
   - Connect: All three from `Set Region Variables`.

5. **Create Three Code Nodes to Process Scan Results**  
   - Names:  
     - `Process EBS Volumes`  
     - `Process Snapshots`  
     - `Process Elastic IPs`  
   - Type: Code  
   - Paste respective JS code for parsing JSON, calculating costs, checking tags, and enriching data (as detailed in the workflow).  
   - Connect each to respective Lambda scan node output.

6. **Create Aggregate Node**  
   - Name: `Aggregate All Resources`  
   - Type: Aggregate  
   - Operation: Aggregate All Item Data  
   - Destination Field Name: `data`  
   - Connect from all three processing nodes.

7. **Create Code Node for Summary Statistics**  
   - Name: `Calculate Summary Stats`  
   - Type: Code  
   - Paste JS code that calculates totals, breakdowns, compliance, and top expensive resources.  
   - Connect from `Aggregate All Resources`.

8. **Create If Node for Resource Found Check**  
   - Name: `If Resources Found`  
   - Type: If  
   - Condition: `{{$json.summary.totalResourcesFound}} > 0`  
   - Connect from `Calculate Summary Stats`.

9. **True Branch: Reporting and Alerts for Found Resources**  
   - Create `Generate HTML Report` (Code node) generating styled HTML summary.  
   - Create `Send Slack Alert` (Slack node) configured with OAuth/Webhook, text using expressions with summary data, Slack channel from config.  
   - Create `Generate CSV Export` (Code node) converting resource data into CSV and structured data.  
   - Create `Send Email Report` (Gmail node) sending HTML report to emails from config, using Gmail OAuth2 credentials.  
   - Create `Log to Google Sheets (Found)` (Google Sheets node) appending/updating row with scan summary and details.  
   - Connect flow:  
     - `If Resources Found` (true) → `Generate HTML Report` → `Send Email Report`  
     - `If Resources Found` (true) → `Send Slack Alert`  
     - `If Resources Found` (true) → `Generate CSV Export` → `Log to Google Sheets (Found)`  

10. **False Branch: Reporting and Alerts for No Resources Found**  
    - Create `None Found Slack Message` (Slack node) sending "zero waste detected" message.  
    - Create `Clean Scan CSV Export` (Code node) generating a single clean scan CSV row.  
    - Create `Append or update row in sheet` (Google Sheets node) logging clean scan data.  
    - Connect flow:  
      - `If Resources Found` (false) → `None Found Slack Message`  
      - `If Resources Found` (false) → `Clean Scan CSV Export` → `Append or update row in sheet`

11. **Create Sticky Notes** to add prerequisites, configuration, processing explanations, and troubleshooting tips for user guidance.

12. **Configure all credentials:**  
    - AWS IAM with EC2 read and Lambda invoke permissions for Lambda nodes.  
    - Slack OAuth/Webhook for Slack nodes.  
    - Gmail OAuth2 for Gmail node.  
    - Google Sheets OAuth2 for Google Sheets nodes.

13. **Test workflow manually:** Run full execution to verify Lambda invocation, data processing, and notifications.

14. **Activate workflow:** Enable for scheduled weekly scans.

---

## 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                              |
|-----------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Lambda function source code and instructions: [https://github.com/chadmcrowell/lambda-function-for-aws-orphaned-resource-scanner](https://github.com/chadmcrowell/lambda-function-for-aws-orphaned-resource-scanner) | Required AWS Lambda function for scanning orphaned resources.                                                |
| Cost assumptions: EBS $0.10/GB-month, Snapshots $0.05/GB-month, Elastic IPs $3.60/month                    | Embedded in processing nodes for cost calculations.                                                         |
| Required AWS IAM permissions: EC2 read-only, Lambda invoke                                                | Prerequisite for Lambda functions to scan resources.                                                        |
| Slack channel and credentials must have posting permissions                                               | For sending alerts to designated channel.                                                                   |
| Gmail OAuth2 with send email permission required                                                         | To send HTML reports by email.                                                                               |
| Google Sheets must have correctly named sheets and headers matching the workflow column mappings          | To log audit trail entries successfully.                                                                    |
| Common troubleshooting tips include checking IAM policy wildcards for Lambda 403 errors and OAuth scopes | Sticky Note3 in workflow.                                                                                     |
| Typical monthly savings range from $50 to $10,000 depending on orphaned resource volume                   | Reported in workflow overview for user expectation management.                                              |
| The workflow is read-only and compliant, no destructive actions unless cleanup enabled via config        | Safety feature to avoid accidental resource deletion.                                                       |

---

# Disclaimer

The text provided derives exclusively from an automated n8n workflow designed for lawful and public AWS resource management. It complies fully with applicable content policies and contains no illegal, offensive, or protected elements.

---