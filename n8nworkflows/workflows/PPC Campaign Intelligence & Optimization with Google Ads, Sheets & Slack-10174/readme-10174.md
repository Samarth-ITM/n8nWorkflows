PPC Campaign Intelligence & Optimization with Google Ads, Sheets & Slack

https://n8nworkflows.xyz/workflows/ppc-campaign-intelligence---optimization-with-google-ads--sheets---slack-10174


# PPC Campaign Intelligence & Optimization with Google Ads, Sheets & Slack

### 1. Workflow Overview

This workflow, titled **PPC Campaign Intelligence & Optimization with Google Ads, Sheets & Slack**, automates the daily monitoring, analysis, and reporting of Google Ads campaigns. It is designed for PPC managers and marketing teams who want data-driven insights, timely alerts, and actionable recommendations to optimize their advertising spend and performance.

The workflow is structured into the following logical blocks:

- **1.1 Scheduled Data Retrieval:** Automatically triggers daily to fetch Google Ads campaign data.
- **1.2 AI-Powered Performance Analysis:** Processes raw campaign metrics to score and categorize campaign performance using custom JavaScript logic.
- **1.3 Performance-Based Routing:** Routes campaigns based on their performance tier to appropriate notification and logging paths.
- **1.4 Data Logging and Dashboard Updates:** Records campaign data in Google Sheets for historical and dashboard tracking.
- **1.5 Alerting System:** Sends Slack notifications to highlight scaling opportunities or performance issues.
- **1.6 Action Plan Generation and Emailing:** Creates personalized email reports with recommendations and action plans, then sends them to the PPC team.
- **1.7 Daily Summary Generation:** Aggregates overall campaign performance metrics into a summary for daily overview.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Data Retrieval

- **Overview:** This block triggers the workflow daily and fetches campaign metrics from Google Ads.
- **Nodes Involved:**  
  - Schedule Daily Check  
  - Fetch Google Ads Data

- **Node Details:**

  - **Schedule Daily Check**  
    - Type: Schedule Trigger  
    - Role: Triggers workflow every 24 hours (daily) at 9 AM (timezone depends on server/local settings).  
    - Configuration: Interval set to 24 hours.  
    - Inputs: None (trigger node).  
    - Outputs: Starts the workflow to Fetch Google Ads Data.  
    - Edge Cases: Missed triggers if n8n server is down; time zone mismatch can affect trigger time.

  - **Fetch Google Ads Data**  
    - Type: Google Ads Node  
    - Role: Pulls campaign performance metrics from Google Ads API.  
    - Configuration: Default request options; requires Google Ads credentials and permissions to read campaign metrics.  
    - Inputs: Trigger from Schedule Daily Check.  
    - Outputs: Raw campaign data JSON to AI Performance Analysis node.  
    - Edge Cases: API quota limits, authentication errors, network timeouts, empty or incomplete data if Google Ads account is empty or restricted.

---

#### 2.2 AI-Powered Performance Analysis

- **Overview:** Applies custom JavaScript to evaluate each campaign’s performance metrics (CTR, conversion rate, cost efficiency, traffic volume), assigning scores, tiers, alerts, insights, and recommendations.
- **Nodes Involved:**  
  - AI Performance Analysis

- **Node Details:**

  - **AI Performance Analysis**  
    - Type: Code Node (JavaScript)  
    - Role: Analyzes metrics to compute a performance score (0-100) and performance tier (Excellent, Good, Fair, Underperforming). Generates insights, recommendations, and alert levels (normal, warning, critical).  
    - Configuration: Embedded JS code with detailed scoring logic based on CTR, conversion rate, cost per conversion, and clicks volume.  
    - Expressions: Uses campaign metrics from input JSON; calculates derived metrics like CTR %, conversion rate %, cost per conversion, avg CPC.  
    - Inputs: Raw campaign data from Fetch Google Ads Data.  
    - Outputs: Enriched campaign data including scores, tiers, alerts, insights, and recommendations to Route by Performance.  
    - Edge Cases: Missing or zero metric values handled gracefully; possible runtime errors if unexpected data format occurs.

---

#### 2.3 Performance-Based Routing

- **Overview:** Routes campaigns into two branches: high performers (Excellent or Good) and others (Fair or Underperforming) for different alerting and logging workflows.
- **Nodes Involved:**  
  - Route by Performance

- **Node Details:**

  - **Route by Performance**  
    - Type: If Node  
    - Role: Checks the `performanceTier` field and routes accordingly.  
    - Configuration: Conditions check if `performanceTier` equals "Excellent" or "Good" for the true branch; else false branch.  
    - Inputs: Enriched campaign data from AI Performance Analysis.  
    - Outputs: Two branches, one for high-performing campaigns and one for others.  
    - Edge Cases: If `performanceTier` is missing or malformed, campaigns default to false branch; no fallback branch explicitly defined.

---

#### 2.4 Data Logging and Dashboard Updates

- **Overview:** Logs campaign data into two Google Sheets tabs: one for updating the daily dashboard and another for maintaining a historical log.
- **Nodes Involved:**  
  - Update Campaign Dashboard  
  - Log All Campaigns

- **Node Details:**

  - **Update Campaign Dashboard**  
    - Type: Google Sheets  
    - Role: Appends or updates daily campaign performance metrics in the "Daily Performance" sheet tab.  
    - Configuration: Maps campaign metrics (CPC, CTR, Cost, Date, Tier, Score, Clicks, Status, etc.) to spreadsheet columns. Requires Google Sheets credentials and spreadsheet/document ID.  
    - Inputs: Campaign data from Route by Performance (both branches).  
    - Outputs: Next node Generate Daily Summary.  
    - Edge Cases: Spreadsheet ID or sheet name misconfiguration; API quota limits; data mapping errors.

  - **Log All Campaigns**  
    - Type: Google Sheets  
    - Role: Appends detailed campaign records to a "Campaign Log" tab for historical tracking.  
    - Configuration: Maps key metrics and timestamps; uses the same spreadsheet as dashboard.  
    - Inputs: Campaign data from Route by Performance (both branches).  
    - Outputs: None directly; parallel logging with dashboard update.  
    - Edge Cases: Same as Update Campaign Dashboard.

---

#### 2.5 Alerting System

- **Overview:** Sends Slack notifications to alert the PPC team about campaigns ready to scale or requiring immediate attention.
- **Nodes Involved:**  
  - Alert: Scale Opportunity  
  - Alert: Issues Detected

- **Node Details:**

  - **Alert: Scale Opportunity**  
    - Type: Slack  
    - Role: Sends a Slack message for campaigns categorized as "Excellent" or "Good", highlighting performance and recommending scaling.  
    - Configuration: Uses Slack webhook with templated message including campaign name, score, key metrics, insights, and recommendations.  
    - Inputs: Campaigns routed as high-performing.  
    - Outputs: None.  
    - Edge Cases: Slack webhook invalid or revoked; message formatting issues.

  - **Alert: Issues Detected**  
    - Type: Slack  
    - Role: Sends Slack alerts for campaigns with performance warnings or critical issues (Fair or Underperforming tiers).  
    - Configuration: Slack webhook with a message detailing alert level, score, and issues.  
    - Inputs: Campaigns routed as others.  
    - Outputs: None.  
    - Edge Cases: Same as above.

---

#### 2.6 Action Plan Generation and Emailing

- **Overview:** Generates personalized email reports with detailed analysis and recommendations based on campaign tier and alert level, then sends these reports to the PPC team.
- **Nodes Involved:**  
  - Generate Action Plan  
  - Email Performance Report

- **Node Details:**

  - **Generate Action Plan**  
    - Type: Code Node (JavaScript)  
    - Role: Creates customized email subjects and bodies depending on performance tier (Excellent/Good, Warning/Critical, or Fair). Sets priority flags and flags for required action or scaling recommendation.  
    - Configuration: Embedded JS constructing multi-line email text with campaign metrics, insights, and recommended actions.  
    - Inputs: Campaign data from Route by Performance branches.  
    - Outputs: Enriched campaign data with emailSubject, emailBody, priority.  
    - Edge Cases: Missing tier or alertLevel defaults to “Fair” and “normal” respectively.

  - **Email Performance Report**  
    - Type: Email Send  
    - Role: Sends the generated email report to the PPC team’s email address.  
    - Configuration: Uses SMTP or email credentials, sets from and to addresses, reply-to, and subject/body dynamically from input JSON.  
    - Inputs: Email content from Generate Action Plan.  
    - Outputs: Next node Generate Daily Summary.  
    - Edge Cases: Email server errors; invalid email addresses; message rejected.

---

#### 2.7 Daily Summary Generation

- **Overview:** Aggregates the daily campaign results into an overall summary including counts of campaigns by tier, total spend, and conversions.
- **Nodes Involved:**  
  - Generate Daily Summary

- **Node Details:**

  - **Generate Daily Summary**  
    - Type: Set Node  
    - Role: Calculates totals and counts for all processed campaigns, formats a summary text for reporting or further use.  
    - Configuration: Uses n8n expressions to compute campaign counts by tier, total spend, and conversions; prepares a formatted multiline summary string.  
    - Inputs: Output of either Update Campaign Dashboard or Email Performance Report.  
    - Outputs: Final summary JSON object (could be used for reporting or notifications if extended).  
    - Edge Cases: Empty input arrays lead to zero counts; numeric conversion errors unlikely but possible if input data malformed.

---

### 3. Summary Table

| Node Name               | Node Type           | Functional Role                                   | Input Node(s)             | Output Node(s)                  | Sticky Note                                                                                       |
|-------------------------|---------------------|-------------------------------------------------|---------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Daily Check     | Schedule Trigger    | Triggers workflow daily at 9 AM                  | None                      | Fetch Google Ads Data           | Triggers workflow automatically every morning at 9 AM daily                                    |
| Fetch Google Ads Data    | Google Ads          | Fetches campaign metrics from Google Ads API     | Schedule Daily Check       | AI Performance Analysis         | Pulls all campaign metrics from Google Ads API                                                 |
| AI Performance Analysis  | Code (JavaScript)   | Scores campaigns based on CTR, conversions, cost | Fetch Google Ads Data      | Route by Performance            | Scores campaigns based on CTR, conversions, and cost efficiency                                |
| Route by Performance     | If                  | Splits campaigns into high-performers vs others  | AI Performance Analysis    | Update Campaign Dashboard, Log All Campaigns, Alert nodes, Generate Action Plan | Splits campaigns into high-performers versus worst ones                                       |
| Update Campaign Dashboard| Google Sheets       | Logs daily metrics to performance tracking tab   | Route by Performance       | Generate Daily Summary          | Logs daily metrics to Google Sheets performance tracking tab                                   |
| Log All Campaigns        | Google Sheets       | Records all campaign data to historical log      | Route by Performance       | None                           | Records all campaign data to historical log spreadsheet                                        |
| Alert: Scale Opportunity | Slack               | Sends Slack notification for scaling opportunities| Route by Performance (True)| None                           | Sends Slack notification for excellent performing campaigns ready to scale                     |
| Alert: Issues Detected   | Slack               | Sends Slack warning for underperforming campaigns| Route by Performance (False)| None                          | Sends Slack warning for underperforming campaigns requiring immediate action                   |
| Generate Action Plan     | Code (JavaScript)   | Creates personalized email reports based on tier| Route by Performance       | Email Performance Report       | Creates customized email reports based on campaign performance tier                            |
| Email Performance Report | Email Send          | Sends email with campaign performance summary    | Generate Action Plan       | Generate Daily Summary          | Sends personalized action plan emails to PPC team members                                     |
| Generate Daily Summary   | Set                 | Aggregates campaign counts and spend summary     | Update Campaign Dashboard, Email Performance Report | None                  | Calculates total spend, conversions, and campaign distribution summary                         |
| Sticky Note              | Sticky Note         | Documentation and comments                        | None                      | None                           | # 📊 PPC Campaign Intelligence System ... (full content in sticky note at workflow start)     |
| Sticky Note1             | Sticky Note         | Comment on schedule trigger                       | None                      | None                           | Triggers workflow automatically every morning at 9 AM daily                                   |
| Sticky Note2             | Sticky Note         | Comment on Google Ads data fetching               | None                      | None                           | Pulls all campaign metrics from Google Ads API                                                |
| Sticky Note3             | Sticky Note         | Comment on AI analysis                            | None                      | None                           | Scores campaigns based on CTR, conversions, and cost efficiency                               |
| Sticky Note4             | Sticky Note         | Comment on routing                                | None                      | None                           | Splits campaigns into high-performers versus worst ones                                      |
| Sticky Note5             | Sticky Note         | Comment on logging                                | None                      | None                           | Records all campaign data to historical log spreadsheet                                       |
| Sticky Note6             | Sticky Note         | Comment on dashboard update                       | None                      | None                           | Logs daily metrics to Google Sheets performance tracking tab                                  |
| Sticky Note7             | Sticky Note         | Comment on Slack alert for scaling                | None                      | None                           | Sends Slack notification for excellent performing campaigns ready to scale                    |
| Sticky Note8             | Sticky Note         | Comment on Slack alert for issues                 | None                      | None                           | Sends Slack warning for underperforming campaigns requiring immediate action                  |
| Sticky Note9             | Sticky Note         | Comment on email reporting                        | None                      | None                           | Creates customized email reports based on campaign performance tier                           |
| Sticky Note10            | Sticky Note         | Comment on email sending                          | None                      | None                           | Sends personalized action plan emails to PPC team members                                    |
| Sticky Note11            | Sticky Note         | Comment on daily summary                          | None                      | None                           | Calculates total spend, conversions, and campaign distribution summary                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Name: `Schedule Daily Check`  
   - Type: Schedule Trigger  
   - Set interval to every 24 hours (daily)  
   - Optional: Adjust time zone or start time to 9 AM local time.

2. **Add a Google Ads node**  
   - Name: `Fetch Google Ads Data`  
   - Connect input from `Schedule Daily Check` node  
   - Configure Google Ads credentials with read access  
   - Set to fetch relevant campaign metrics (impressions, clicks, cost_micros, conversions, etc.)  
   - No special request options needed.

3. **Add a Code node (JavaScript)**  
   - Name: `AI Performance Analysis`  
   - Connect input from `Fetch Google Ads Data` node  
   - Paste the provided JavaScript code that calculates performance scores, tiers, alert levels, insights, and recommendations based on campaign metrics.  
   - Ensure it handles missing or zero values gracefully.

4. **Add an If node**  
   - Name: `Route by Performance`  
   - Connect input from `AI Performance Analysis` node  
   - Configure condition: check if `performanceTier` equals "Excellent" OR "Good"  
   - True branch: high performers; False branch: others.

5. **Add Google Sheets node for dashboard update**  
   - Name: `Update Campaign Dashboard`  
   - Connect input from both branches (True and False) of `Route by Performance`  
   - Configure Google Sheets credentials  
   - Set operation to append or update in sheet named "Daily Performance"  
   - Map columns: CPC, CTR, Cost, Date (current date), Tier, Score, Clicks, Status, Conv Rate, Cost/Conv, Conversions, Impressions  
   - Use your spreadsheet document ID (replace placeholder).

6. **Add Google Sheets node for historical logging**  
   - Name: `Log All Campaigns`  
   - Connect input from both branches of `Route by Performance`  
   - Configure to append or update in sheet named "Campaign Log"  
   - Map columns: Cost, Tier, Score, Campaign, Cost/Conv, Timestamp (current ISO timestamp), Alert Level, Conversions  
   - Use same Google Sheets credentials and spreadsheet ID.

7. **Add Slack nodes for alerts**  
   - Name: `Alert: Scale Opportunity`  
   - Connect input from True branch of `Route by Performance`  
   - Configure Slack webhook URL for your scaling opportunity channel  
   - Use templated message including campaign name, score, tier, metrics, insights, and recommendations.

   - Name: `Alert: Issues Detected`  
   - Connect input from False branch of `Route by Performance`  
   - Configure Slack webhook URL for your alert/warning channel  
   - Use templated message with alert level, score, tier, metrics, and issues.

8. **Add a Code node for generating email action plans**  
   - Name: `Generate Action Plan`  
   - Connect input from both branches of `Route by Performance` (parallel or merged if needed)  
   - Paste provided JavaScript code that generates customized email subjects and bodies depending on performance tier and alert level.

9. **Add an Email Send node**  
   - Name: `Email Performance Report`  
   - Connect input from `Generate Action Plan`  
   - Configure SMTP/email credentials  
   - Set recipient as `ppc-team@yourcompany.com` or relevant email  
   - Use dynamic subject and body from input JSON fields (`emailSubject` and `emailBody`)  
   - Set `From` and `Reply-To` accordingly.

10. **Add a Set node for daily summary**  
    - Name: `Generate Daily Summary`  
    - Connect input from both `Update Campaign Dashboard` and `Email Performance Report` nodes (both paths lead here)  
    - Create fields calculating: total campaigns, counts by tier (Excellent, Good, Fair, Underperforming), total spend, total conversions  
    - Format a multi-line summary string for reporting or further notifications.

11. **(Optional) Add Sticky Note nodes**  
    - For documentation and comments, add Sticky Notes near related nodes to describe their purpose, as per the original workflow’s notes.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                | Context or Link                                                    |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------|
| This workflow was built by Daniel Shashko and connects Google Ads, Google Sheets, Slack, and SMTP/email to automate PPC campaign monitoring and reporting.                   | [LinkedIn Profile](https://www.linkedin.com/in/daniel-shashko/)  |
| The AI analysis node implements a custom scoring system for campaign evaluation based on key PPC metrics with configurable thresholds in the JavaScript code.                | Configuration detail in AI Performance Analysis code node.       |
| Make sure to update Google Sheets document IDs and sheet names to match your environment before running the workflow.                                                      | Google Sheets nodes configuration.                               |
| Slack webhooks must be configured for the relevant channels to receive alerts; ensure webhook URLs are valid and active.                                                   | Slack alert nodes configuration.                                 |
| Email credentials must have permission to send emails and use correct SMTP settings; verify email addresses and reply-to fields are correct.                                | Email Performance Report node configuration.                     |
| This workflow helps avoid missed scaling opportunities and detects underperforming campaigns early by automating daily PPC data analysis and communication.                 | General workflow benefit.                                        |

---

*Disclaimer:* The provided text is exclusively derived from an automated n8n workflow. It complies strictly with content policies and contains no illegal or offensive elements. All data processed is legal and publicly accessible.