Monitor Supply Chain Risks with ScrapeGraphAI Alerts via Slack and Email

https://n8nworkflows.xyz/workflows/monitor-supply-chain-risks-with-scrapegraphai-alerts-via-slack-and-email-6569


# Monitor Supply Chain Risks with ScrapeGraphAI Alerts via Slack and Email

### 1. Workflow Overview

This workflow, titled **"Monitor Supply Chain Risks with ScrapeGraphAI Alerts via Slack and Email"**, automates the daily monitoring of supply chain risks by scraping supplier websites and industry news, analyzing risk levels, and sending alerts or reports through Slack and email.

The workflow is designed for procurement teams and supply chain managers who need continuous oversight of supplier health and industry risks to ensure business continuity and proactive risk mitigation.

**Logical Blocks:**

- **1.1 Daily Trigger:** Initiates the workflow every day at 9:00 AM.
- **1.2 Supplier and Industry Data Scraping:** Collects health and risk indicators from multiple supplier websites and industry news sources using ScrapeGraphAI nodes.
- **1.3 Risk Analysis and Scoring:** Aggregates scraped data, computes risk scores per supplier, and generates an overall risk report.
- **1.4 Alternative Supplier Search:** Finds backup suppliers in the manufacturing category for contingency planning.
- **1.5 Alert Decision Logic:** Determines if high-risk conditions are met to trigger immediate alerts or send daily summaries.
- **1.6 Notifications Dispatch:** Sends Slack alerts and/or emails to procurement teams and posts daily supply chain health reports to Slack channels.

---

### 2. Block-by-Block Analysis

#### 2.1 Daily Trigger

- **Overview:**  
This block triggers the entire risk monitoring workflow automatically at a scheduled time every day.

- **Nodes Involved:**  
  - Daily Risk Check (Schedule Trigger)

- **Node Details:**

  - **Daily Risk Check**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts the workflow at a specified schedule.  
    - *Configuration:* Cron expression set to trigger at 9:00 AM daily (`0 9 * * *`).  
    - *Inputs:* None (trigger node)  
    - *Outputs:* Initiates multiple parallel branches for supplier scraping.  
    - *Edge Cases:* Misconfiguration of cron expression; timezone mismatch causing off-hour triggers.  
    - *Sticky Note:* Step 1 Note explains the scheduling purpose and benefits.

#### 2.2 Supplier and Industry Data Scraping

- **Overview:**  
This block scrapes structured health and risk data from supplier websites and industry news sources using AI-powered scraping.

- **Nodes Involved:**  
  - Scrape Supplier 1  
  - Scrape Supplier 2  
  - Scrape Industry News

- **Node Details:**

  - **Scrape Supplier 1**  
    - *Type:* ScrapeGraphAI  
    - *Role:* Extracts health indicators from the first supplier’s website.  
    - *Configuration:* Uses a user prompt to extract fields like financial status, operational issues, risk score, and source URL from `https://example-supplier-1.com/news`.  
    - *Inputs:* Trigger from Daily Risk Check  
    - *Outputs:* JSON object with supplier health data to Risk Scorer.  
    - *Edge Cases:* Website structure changes, AI extraction errors, network failures.

  - **Scrape Supplier 2**  
    - *Type:* ScrapeGraphAI  
    - *Role:* Similar extraction for second supplier from `https://example-supplier-2.com/investor-relations`.  
    - *Configuration:* Same schema and prompt structure as Supplier 1.  
    - *Inputs:* Trigger from Daily Risk Check  
    - *Outputs:* JSON with supplier 2 data to Risk Scorer.  
    - *Edge Cases:* Same as Supplier 1.

  - **Scrape Industry News**  
    - *Type:* ScrapeGraphAI  
    - *Role:* Scrapes news articles on supply chain disruptions and related risks from Reuters supply chain section.  
    - *Configuration:* Extracts headline, summary, impact level, affected suppliers, etc., from `https://www.reuters.com/business/supply-chain/`.  
    - *Inputs:* Trigger from Daily Risk Check  
    - *Outputs:* News data to Risk Scorer.  
    - *Edge Cases:* News site layout changes, rate limiting, extraction errors.

- *Sticky Note:* Step 2 Note explains the data sources and monitored indicators.

#### 2.3 Risk Analysis and Scoring

- **Overview:**  
Aggregates all scraped supplier and news data to compute risk scores, classify suppliers by risk level, and generate a comprehensive risk report.

- **Nodes Involved:**  
  - Risk Scorer (Code node)

- **Node Details:**

  - **Risk Scorer**  
    - *Type:* Code (JavaScript)  
    - *Role:* Processes inputs from all scraping nodes, calculates risk scores based on financial status, operational issues, regulatory problems, and news sentiment. Generates a summary report with counts of high, medium, and low-risk suppliers.  
    - *Key Logic:*  
      - Assigns weighted points for each risk factor.  
      - Caps scores at 10.  
      - Classifies risk levels: LOW (<3), MEDIUM (3-7), HIGH (8+).  
      - Formats a markdown-style risk report.  
    - *Inputs:* Aggregated JSON from Scrape Supplier 1, Supplier 2, and Industry News.  
    - *Outputs:* JSON with processed suppliers array, summary report string, risk counts, and timestamp.  
    - *Edge Cases:* Missing or malformed data, unexpected JSON structure, runtime errors in JS code.  
    - *Version:* Uses typeVersion 2 for Code node.  
    - *Sticky Note:* Step 3 Note describes scoring factors and output.

#### 2.4 Alternative Supplier Search

- **Overview:**  
Searches for backup suppliers in the manufacturing category to provide alternatives when high-risk suppliers are identified.

- **Nodes Involved:**  
  - Find Alternatives (ScrapeGraphAI)

- **Node Details:**

  - **Find Alternatives**  
    - *Type:* ScrapeGraphAI  
    - *Role:* Scrapes ThomasNet suppliers in manufacturing, extracting company details, certifications, capacity, quality rating, and contact info.  
    - *Configuration:* Uses a user prompt with a JSON schema for company information, scraping from `https://www.thomasnet.com/suppliers/manufacturing`.  
    - *Inputs:* Triggered after Risk Scorer node output.  
    - *Outputs:* Potential supplier alternatives to Check Alert Conditions node.  
    - *Edge Cases:* Website structure changes, missing data, scraping limits.  
    - *Sticky Note:* Step 4 Note explains criteria and benefits.

#### 2.5 Alert Decision Logic

- **Overview:**  
Decides whether to send an immediate alert or a daily summary report based on the number of high-risk suppliers detected.

- **Nodes Involved:**  
  - Check Alert Conditions (If node)  
  - Format Alert (Set node)  
  - Format Daily Report (Set node)

- **Node Details:**

  - **Check Alert Conditions**  
    - *Type:* If node  
    - *Role:* Checks if the count of high-risk suppliers (`high_risk_count`) is greater than 0.  
    - *Conditions:* Logical OR with condition: `high_risk_count > 0`.  
    - *Inputs:* From both Risk Scorer and Find Alternatives nodes (main input merged).  
    - *Outputs:*  
      - True branch leads to Format Alert (for immediate alerts).  
      - False branch leads to Format Daily Report (for daily summary).  
    - *Edge Cases:* Missing or zero `high_risk_count`, ensuring correct data flow.

  - **Format Alert**  
    - *Type:* Set node  
    - *Role:* Prepares an alert message with priority level and actionable recommendations.  
    - *Configuration:*  
      - Creates `alert_message` with markdown including summary report, recommended actions, next steps, and timestamp.  
      - Sets `priority` string: "CRITICAL" if `high_risk_count >= 2`, else "HIGH".  
    - *Inputs:* True branch from Check Alert Conditions.  
    - *Outputs:* To Slack and Email nodes.  
    - *Edge Cases:* String interpolation errors, missing summary data.  
    - *Version:* TypeVersion 3.4.  
    - *Sticky Note:* Step 5 Note explains alert conditions and routing logic.

  - **Format Daily Report**  
    - *Type:* Set node  
    - *Role:* Creates a daily summary message for supply chain health status.  
    - *Configuration:* Contains a `daily_summary` string with summary report, status confirmation, trend analysis, and next check time.  
    - *Inputs:* False branch from Check Alert Conditions.  
    - *Outputs:* To Slack daily report node.  
    - *Edge Cases:* Missing summary data.  
    - *Version:* TypeVersion 3.4.

#### 2.6 Notifications Dispatch

- **Overview:**  
Sends formatted alerts or daily reports to appropriate Slack channels and emails the procurement team with JSON attachments.

- **Nodes Involved:**  
  - Send Slack Alert (Slack node)  
  - Email Procurement Team (Email Send node)  
  - Send Daily Report (Slack node)

- **Node Details:**

  - **Send Slack Alert**  
    - *Type:* Slack node  
    - *Role:* Posts urgent alerts to the `#procurement-alerts` Slack channel.  
    - *Configuration:*  
      - Uses OAuth2 authentication.  
      - Message text set from `alert_message` variable.  
      - Channel specified by name `#procurement-alerts`.  
    - *Inputs:* From Format Alert node.  
    - *Outputs:* None (end node).  
    - *Edge Cases:* Slack API rate limits, authentication failures, invalid channel.

  - **Email Procurement Team**  
    - *Type:* Email Send node  
    - *Role:* Emails detailed supply chain risk data to procurement team.  
    - *Configuration:*  
      - Attaches JSON file containing filtered suppliers with medium and high risk.  
      - Assumes SMTP or configured email credentials available.  
    - *Inputs:* From Format Alert node.  
    - *Outputs:* None (end node).  
    - *Edge Cases:* SMTP errors, attachment size limits.

  - **Send Daily Report**  
    - *Type:* Slack node  
    - *Role:* Posts daily supply chain health summary to `#supply-chain-updates` Slack channel.  
    - *Configuration:*  
      - OAuth2 authentication.  
      - Text set from `daily_summary` variable.  
      - Channel specified by name.  
    - *Inputs:* From Format Daily Report node.  
    - *Outputs:* None.  
    - *Edge Cases:* Same as Send Slack Alert.

- *Sticky Note:* Step 6 Note describes multi-channel notifications, message formatting, and audience targeting.

---

### 3. Summary Table

| Node Name            | Node Type             | Functional Role                        | Input Node(s)                       | Output Node(s)                        | Sticky Note                                                                                   |
|----------------------|-----------------------|-------------------------------------|-----------------------------------|-------------------------------------|----------------------------------------------------------------------------------------------|
| Daily Risk Check      | Schedule Trigger       | Starts daily workflow at 9:00 AM     | None                              | Scrape Supplier 1, Scrape Supplier 2, Scrape Industry News | Step 1 Note: Daily trigger configuration and benefits                                       |
| Scrape Supplier 1    | ScrapeGraphAI          | Scrapes supplier 1 website data      | Daily Risk Check                  | Risk Scorer                         | Step 2 Note: Supplier data scraping details                                                  |
| Scrape Supplier 2    | ScrapeGraphAI          | Scrapes supplier 2 website data      | Daily Risk Check                  | Risk Scorer                         | Step 2 Note                                                                                   |
| Scrape Industry News | ScrapeGraphAI          | Scrapes industry news articles       | Daily Risk Check                  | Risk Scorer                         | Step 2 Note                                                                                   |
| Risk Scorer          | Code                  | Aggregates and scores risk data      | Scrape Supplier 1,2, Industry News | Find Alternatives                   | Step 3 Note: Risk factors and scoring logic                                                 |
| Find Alternatives    | ScrapeGraphAI          | Finds backup suppliers                | Risk Scorer                      | Check Alert Conditions             | Step 4 Note: Alternative supplier search criteria                                           |
| Check Alert Conditions | If                    | Decides alert vs daily report        | Risk Scorer, Find Alternatives   | Format Alert (true), Format Daily Report (false) | Step 5 Note: Alert conditions and decision logic                                            |
| Format Alert         | Set                    | Creates urgent alert message          | Check Alert Conditions (true)     | Send Slack Alert, Email Procurement Team | Step 5 Note                                                                                   |
| Format Daily Report  | Set                    | Creates daily summary message         | Check Alert Conditions (false)    | Send Daily Report                  |                                                                                              |
| Send Slack Alert     | Slack                  | Sends alert to procurement Slack     | Format Alert                     | None                              | Step 6 Note: Multi-channel notification details                                              |
| Email Procurement Team | Email Send            | Emails detailed risk data             | Format Alert                     | None                              | Step 6 Note                                                                                   |
| Send Daily Report    | Slack                  | Sends daily report to Slack channel   | Format Daily Report              | None                              | Step 6 Note                                                                                   |
| Step 1 Note          | Sticky Note            | Documentation for daily trigger       | None                              | None                              |                                                                                        |
| Step 2 Note          | Sticky Note            | Documentation for supplier scraping   | None                              | None                              |                                                                                        |
| Step 3 Note          | Sticky Note            | Documentation for risk scoring logic  | None                              | None                              |                                                                                        |
| Step 4 Note          | Sticky Note            | Documentation for alternative supplier search | None                         | None                              |                                                                                        |
| Step 5 Note          | Sticky Note            | Documentation for alert logic          | None                              | None                              |                                                                                        |
| Step 6 Note          | Sticky Note            | Documentation for notifications        | None                              | None                              |                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node ("Daily Risk Check")**  
   - Set type to Schedule Trigger.  
   - Configure with a cron expression: `0 9 * * *` (runs daily at 9:00 AM).  
   - No inputs. Outputs to three nodes (Scrape Supplier 1, 2, Industry News).

2. **Add ScrapeGraphAI Nodes for Data Collection**  
   - Create three separate ScrapeGraphAI nodes:  
     - "Scrape Supplier 1" configured with user prompt for supplier health schema; URL: `https://example-supplier-1.com/news`.  
     - "Scrape Supplier 2" with same schema; URL: `https://example-supplier-2.com/investor-relations`.  
     - "Scrape Industry News" with news article schema; URL: `https://www.reuters.com/business/supply-chain/`.  
   - Connect all three nodes’ inputs to "Daily Risk Check" output.

3. **Create a Code Node ("Risk Scorer")**  
   - Set node type to Code, JavaScript mode.  
   - Take all outputs from the three ScrapeGraphAI nodes as inputs (merge).  
   - Paste the provided JS code that:  
     - Aggregates supplier data.  
     - Calculates risk scores and assigns risk levels.  
     - Formats a markdown summary report.  
   - Output JSON containing suppliers array, summary report, risk counts, and timestamp.  
   - Connect outputs of all ScrapeGraphAI nodes to this node.

4. **Add ScrapeGraphAI Node for Alternative Supplier Search ("Find Alternatives")**  
   - Configure with user prompt for alternative suppliers schema.  
   - URL: `https://www.thomasnet.com/suppliers/manufacturing`.  
   - Connect input from "Risk Scorer" output.

5. **Insert an If Node ("Check Alert Conditions")**  
   - Configure condition: `high_risk_count > 0`.  
   - Connect inputs from both "Risk Scorer" and "Find Alternatives" outputs (merge as needed).  
   - True branch connects to "Format Alert".  
   - False branch connects to "Format Daily Report".

6. **Create a Set Node ("Format Alert")**  
   - Assign two fields:  
     - `alert_message`: Markdown string with urgent risk summary, recommended actions, next steps, and timestamp.  
     - `priority`: "CRITICAL" if `high_risk_count >= 2` else "HIGH".  
   - Connect true branch of "Check Alert Conditions" here.

7. **Create a Set Node ("Format Daily Report")**  
   - Assign a `daily_summary` string field with daily supply chain health summary message, trend analysis, and next check info.  
   - Connect false branch of "Check Alert Conditions" here.

8. **Add Slack Node ("Send Slack Alert")**  
   - Configure with OAuth2 credentials.  
   - Set channel name to `#procurement-alerts`.  
   - Message text set from `alert_message` variable.  
   - Connect output of "Format Alert" to this node.

9. **Add Email Send Node ("Email Procurement Team")**  
   - Configure with SMTP or relevant email credentials.  
   - Attach JSON file containing filtered high and medium risk suppliers from "Risk Scorer" output.  
   - Connect output of "Format Alert" here in parallel with Slack alert.

10. **Add Slack Node ("Send Daily Report")**  
    - Configure with OAuth2 credentials.  
    - Set channel name to `#supply-chain-updates`.  
    - Message text set from `daily_summary` variable.  
    - Connect output of "Format Daily Report" here.

11. **Add Sticky Notes for Documentation (Optional but Recommended)**  
    - Add notes per step explaining purpose, configuration, and benefits as in the original workflow.

12. **Verify Credentials**  
    - Setup OAuth2 credentials for Slack nodes with access to specified channels.  
    - Setup email credentials (SMTP or other) for sending emails.

13. **Test and Activate Workflow**  
    - Run manual test or wait for scheduled trigger.  
    - Verify data scraping, risk scoring, alert formatting, and notifications work as expected.  
    - Monitor logs for errors such as scraping failures or API errors.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                        |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| Step 1-6 sticky notes provide detailed explanations of each workflow stage, including scheduling, data scraping, scoring, alternative search, alerting, and notifications. | Internal workflow documentation                        |
| Slack channel names used: `#procurement-alerts` for urgent alerts, `#supply-chain-updates` for daily reports.                                                    | Slack workspace configuration                          |
| Email node sends detailed JSON attachments of medium and high-risk suppliers for further analysis by procurement teams.                                         | Email communication with procurement                    |
| ScrapeGraphAI nodes leverage AI-powered scraping based on user prompts and schemas to extract structured data from supplier and news websites.                  | ScrapeGraphAI integration details                       |
| Risk scores are capped at 10 and classified as LOW (<3), MEDIUM (3-7), HIGH (8+), providing a clear risk prioritization framework.                              | Risk scoring methodology                                |

---

This document fully describes the workflow structure, logic, nodes, configuration, and instructions to reproduce the workflow with clarity and precision for advanced users or AI agents.