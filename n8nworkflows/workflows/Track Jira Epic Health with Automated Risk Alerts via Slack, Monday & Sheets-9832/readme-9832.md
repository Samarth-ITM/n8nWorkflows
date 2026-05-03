Track Jira Epic Health with Automated Risk Alerts via Slack, Monday & Sheets

https://n8nworkflows.xyz/workflows/track-jira-epic-health-with-automated-risk-alerts-via-slack--monday---sheets-9832


# Track Jira Epic Health with Automated Risk Alerts via Slack, Monday & Sheets

### 1. Workflow Overview

This workflow, titled **"Jira Epic Health Score & Risk Dashboard"**, is designed to monitor the health status of Jira Epics on a scheduled basis, identify potential risks, and notify relevant teams while updating multiple platforms for centralized risk management and tracking.

**Target Use Cases:**  
- Agile teams managing multiple Jira Epics to maintain visibility on project health  
- Automated risk detection and alerting for Epics that are potentially problematic  
- Synchronizing health metrics across collaboration tools (Slack, Monday.com, Google Sheets)  
- Maintaining historical records and dashboards for trend analysis and compliance  

**Logical Blocks:**  
- **1.1 Scheduled Trigger:** Periodic initiation every 6 hours.  
- **1.2 Epic Retrieval:** Fetch all Epics from Jira and split for individual processing.  
- **1.3 Epic and Linked Issues Fetch:** Retrieve detailed information for each Epic and its linked issues.  
- **1.4 Health Score Calculation:** Compute a weighted health score based on issue metrics.  
- **1.5 Risk Assessment:** Determine if Epic is at risk via threshold check.  
- **1.6 At-Risk Epic Handling:** Update Jira Epic labels, alert Slack, and log to Monday.com and Google Sheets.  

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  Initiates the workflow automatically every 6 hours to ensure continual monitoring of Jira Epics.

- **Nodes Involved:**  
  - *Trigger Every 6 Hours* (Cron Node)  
  - *📊 Workflow Overview* (Sticky Note)

- **Node Details:**  

  - **Trigger Every 6 Hours**  
    - *Type:* Cron Trigger  
    - *Configuration:* Runs every 6 hours (fixed interval)  
    - *Input/Output:* No inputs; outputs trigger event to start workflow  
    - *Potential Failures:* Cron misconfiguration could prevent execution; system downtime affects trigger.  
    - *Notes:* Reliable scheduling is critical for timely health checks.

  - **📊 Workflow Overview**  
    - *Type:* Sticky Note  
    - *Purpose:* Provides a high-level explanation of workflow start and purpose.  
    - *Input/Output:* Informational only; no connections.

#### 1.2 Epic Retrieval

- **Overview:**  
  Fetches all Epics from Jira using the Jira REST API and splits them into individual items for detailed processing.

- **Nodes Involved:**  
  - *Fetch All Epics from Jira* (Jira Node)  
  - *Split Epics for Processing* (Function Node)  
  - *📥 Fetch Epics* (Sticky Note)  
  - *🔀 Split Epics* (Sticky Note)

- **Node Details:**  

  - **Fetch All Epics from Jira**  
    - *Type:* Jira Node (getAll operation)  
    - *Configuration:* Retrieves all issues filtered to Epics; uses Jira Software Cloud credentials  
    - *Key Expressions:* None specific; returns full list of Epics  
    - *Input:* Trigger node output  
    - *Output:* JSON array of all Epics with keys and fields  
    - *Failures:* Possible auth failure, API rate limiting, network errors  
    - *Credential:* Jira SW Cloud account (vivek)  

  - **Split Epics for Processing**  
    - *Type:* Function Node  
    - *Configuration:* Iterates over all Epics; outputs each Epic key as a separate item  
    - *Key Expressions:* `item.json.key` extracted and output as `epicKey`  
    - *Input:* List of Epics from Jira node  
    - *Output:* Array of individual Epic objects for sequential processing  
    - *Failures:* Code errors if input malformed; empty Epic list edge case handled implicitly  
   
  - **📥 Fetch Epics & 🔀 Split Epics**  
    - *Type:* Sticky Notes  
    - *Purpose:* Document intent and process of fetching and splitting Epics for clarity.

#### 1.3 Epic and Linked Issues Fetch

- **Overview:**  
  For each Epic key, fetches detailed Epic metadata and all linked child issues to gather data necessary for health score calculation.

- **Nodes Involved:**  
  - *Fetch Epic + Linked Issues* (Jira Node)  
  - *🔗 Fetch Linked* (Sticky Note)

- **Node Details:**  

  - **Fetch Epic + Linked Issues**  
    - *Type:* Jira Node (get operation)  
    - *Configuration:* Uses dynamic Epic key from previous function (`={{ $json.epicKey }}`)  
    - *Fields Retrieved:* Epic metadata and linked issues with their status, type, labels, cycle times, custom fields  
    - *Input:* Individual Epic key from split function  
    - *Output:* JSON object containing Epic and linked issues details  
    - *Failures:* API call failures, missing linked issues, permissions errors  
    - *Credential:* Jira SW Cloud account (vivek)  

  - **🔗 Fetch Linked**  
    - *Type:* Sticky Note  
    - *Purpose:* Explains the nature and scope of data fetched for each Epic.

#### 1.4 Health Score Calculation

- **Overview:**  
  Calculates a weighted health score for each Epic based on linked issue metrics such as cycle time, bug ratio, churn, and blockers.

- **Nodes Involved:**  
  - *Calculate Health Score* (Function Node)  
  - *📈 Health Score* (Sticky Note)

- **Node Details:**  

  - **Calculate Health Score**  
    - *Type:* Function Node  
    - *Logic:*  
      - Extracts linked issues from input  
      - Computes average cycle time, counts blockers, churn (reopened), bugs  
      - Calculates ratios per total issues  
      - Applies weighted formula: 40% avgCycleTime, 30% bugRatio, 20% churnRatio, 10% blockerRatio  
      - Outputs Epic key and health score rounded to 2 decimals  
    - *Input:* JSON with Epic and linked issues  
    - *Output:* JSON with `epicKey` and `healthScore`  
    - *Failures:* Logic errors if issue data missing; division by zero mitigated by default total=1  
    - *Edge Cases:* Epics without linked issues; no blockers or bugs  
   
  - **📈 Health Score**  
    - *Type:* Sticky Note  
    - *Purpose:* Documents scoring algorithm, weighting, and risk threshold interpretation.

#### 1.5 Risk Assessment

- **Overview:**  
  Determines if the Epic’s health score exceeds the risk threshold (0.6) and routes execution accordingly.

- **Nodes Involved:**  
  - *Is Score > 0.6 (At Risk)?* (If Node)  
  - *⚠️ Risk Check* (Sticky Note)

- **Node Details:**  

  - **Is Score > 0.6 (At Risk)?**  
    - *Type:* If Node (condition check)  
    - *Condition:* Checks if `healthScore` is greater than 0.6  
    - *Input:* Health score JSON from previous node  
    - *Output:* Two branches: True (At Risk) and False (Healthy)  
    - *Failures:* Edge cases if healthScore missing or non-numeric  
   
  - **⚠️ Risk Check**  
    - *Type:* Sticky Note  
    - *Purpose:* Explains risk interpretation and subsequent workflow actions.

#### 1.6 At-Risk Epic Handling

- **Overview:**  
  For Epics flagged as at-risk, updates Jira labels, sends Slack alerts, updates Monday.com pulse, and logs to Google Sheets. Healthy Epics update only Monday.com and Sheets without Jira or Slack changes.

- **Nodes Involved:**  
  - *Update Epic in Jira* (Jira Node)  
  - *Alert Slack Channel* (Slack Node)  
  - *Update Monday.com Pulse* (Monday.com Node)  
  - *Log to Google Sheets* (Google Sheets Node)  
  - *🔄 Jira Update* (Sticky Note)  
  - *🚨 Slack Alert* (Sticky Note)  
  - *📋 Monday Pulse* (Sticky Note)  
  - *📊 Sheets Log* (Sticky Note)

- **Node Details:**  

  - **Update Epic in Jira**  
    - *Type:* Jira Node (update operation)  
    - *Configuration:* Updates Epic with label "At Risk" and sets custom field `customfield_10060` to health score  
    - *Input:* Epic key and health score from risk check (True branch)  
    - *Failures:* Permission errors, field ID misconfiguration, API errors  
    - *Credential:* Jira SW Cloud account (vivek)  

  - **Alert Slack Channel**  
    - *Type:* Slack Node  
    - *Configuration:* Posts message to `#project-alerts` channel with Epic key, health score, and URL  
    - *Input:* Epic key and health score (True branch)  
    - *Failures:* Slack API auth errors, invalid channel  
    - *Credential:* Slack API credential (placeholder name: Your Slack Credential)  

  - **Update Monday.com Pulse**  
    - *Type:* Monday.com Node (update operation)  
    - *Configuration:* Updates board item with health score and risk status  
    - *Input:* Epic data (both True and False branches)  
    - *Failures:* Monday API failures, credential errors  
    - *Credential:* Monday.com API credential (Your Monday Credential)  

  - **Log to Google Sheets**  
    - *Type:* Google Sheets Node (append operation)  
    - *Configuration:* Appends row with timestamp, Epic key, and health score to specified sheet and range (A:C)  
    - *Input:* Epic data (both True and False branches)  
    - *Failures:* Authentication errors, sheet ID misconfiguration  
    - *Credential:* Google API credential (Your Google Credential)  

  - **Sticky Notes**  
    - *🔄 Jira Update:* Explains the Jira update process and impact on Epic view and filtering  
    - *🚨 Slack Alert:* Details Slack message content and importance  
    - *📋 Monday Pulse:* Describes Monday.com synchronization and data requirements  
    - *📊 Sheets Log:* Describes logging purpose and data columns  

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                     | Input Node(s)                | Output Node(s)                    | Sticky Note                                                                                                   |
|----------------------------|---------------------|-----------------------------------|-----------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Trigger Every 6 Hours       | Cron Trigger        | Scheduled workflow initiation     | —                           | Fetch All Epics from Jira        | 📊 WORKFLOW START: Runs every 6 hours to monitor Jira Epics                                                  |
| 📊 Workflow Overview       | Sticky Note         | Workflow start documentation      | —                           | —                               | 📊 WORKFLOW START: Overview of workflow purpose and steps                                                    |
| Fetch All Epics from Jira  | Jira Node           | Retrieve all Epics from Jira      | Trigger Every 6 Hours        | Split Epics for Processing       | 📥 EPIC RETRIEVAL: Uses Jira API to fetch all Epics                                                          |
| 📥 Fetch Epics             | Sticky Note         | Epic retrieval explanation        | —                           | —                               | 📥 EPIC RETRIEVAL: Details on fetching Epics                                                                |
| Split Epics for Processing | Function Node       | Split Epic list into singles      | Fetch All Epics from Jira    | Fetch Epic + Linked Issues       | 🔀 SPLIT EPICS: Explains splitting batch into individual Epic objects                                        |
| 🔀 Split Epics             | Sticky Note         | Split logic explanation           | —                           | —                               | 🔀 SPLIT EPICS: Explains purpose of splitting for detailed processing                                        |
| Fetch Epic + Linked Issues | Jira Node           | Fetch Epic details and linked issues | Split Epics for Processing   | Calculate Health Score           | 🔗 FETCH LINKED: Explains fetching Epic metadata and linked issues                                           |
| 🔗 Fetch Linked            | Sticky Note         | Linked issues fetching explanation | —                           | —                               | 🔗 FETCH LINKED: Details of data retrieved for health metrics                                                |
| Calculate Health Score     | Function Node       | Compute health score per Epic     | Fetch Epic + Linked Issues   | Is Score > 0.6 (At Risk)?        | 📈 HEALTH SCORE: Documents weighted health score formula                                                     |
| 📈 Health Score            | Sticky Note         | Health score algorithm description | —                           | —                               | 📈 HEALTH SCORE: Details on metrics and scoring thresholds                                                  |
| Is Score > 0.6 (At Risk)? | If Node             | Risk threshold decision           | Calculate Health Score       | Update Epic in Jira, Alert Slack Channel (True branch)  | ⚠️ RISK THRESHOLD: Explains risk condition and actions                                                      |
|                            |                     |                                   |                             | Update Monday.com Pulse, Log to Google Sheets (False branch) |                                                                                                              |
| ⚠️ Risk Check              | Sticky Note         | Risk check explanation            | —                           | —                               | ⚠️ RISK THRESHOLD: Explains risk interpretation and next steps                                              |
| Update Epic in Jira        | Jira Node           | Label at-risk Epic and update field | Is Score > 0.6 (At Risk)? (True) | Alert Slack Channel            | 🔄 JIRA UPDATE: Explains updating Epic with label and health score                                          |
| 🔄 Jira Update             | Sticky Note         | Jira update details               | —                           | —                               | 🔄 JIRA UPDATE: Describes Jira impact and audit trail                                                       |
| Alert Slack Channel        | Slack Node          | Notify team of at-risk Epic       | Update Epic in Jira          | —                               | 🚨 SLACK ALERT: Details Slack message content and channel                                                  |
| 🚨 Slack Alert             | Sticky Note         | Slack alert explanation           | —                           | —                               | 🚨 SLACK ALERT: Documents alert contents and urgency                                                       |
| Update Monday.com Pulse    | Monday.com Node     | Update Monday.com board           | Is Score > 0.6 (At Risk)? (Both branches) | Log to Google Sheets          | 📋 MONDAY.COM PULSE: Explains Monday.com synchronization and fields                                        |
| 📋 Monday Pulse            | Sticky Note         | Monday.com update explanation     | —                           | —                               | 📋 MONDAY.COM PULSE: Details on pulse update and credential requirements                                   |
| Log to Google Sheets       | Google Sheets Node  | Append Epic health data to sheet  | Update Monday.com Pulse      | —                               | 📊 GOOGLE SHEETS LOG: Documents columns and usage for historical tracking                                  |
| 📊 Sheets Log              | Sticky Note         | Google Sheets logging explanation | —                           | —                               | 📊 GOOGLE SHEETS LOG: Details on data tracked for analysis and dashboards                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Cron Trigger Node**  
   - Type: Cron  
   - Name: "Trigger Every 6 Hours"  
   - Set to trigger every 6 hours (mode: everyX, value: 6)  
   - This node starts the workflow on schedule.

2. **Add Jira Node to Fetch All Epics**  
   - Type: Jira  
   - Name: "Fetch All Epics from Jira"  
   - Operation: `getAll` (retrieve all issues)  
   - Configure credentials with Jira Software Cloud API (OAuth or API token)  
   - Filter to Epics only via Jira query or post-filtering (if needed)  
   - Connect trigger node output to this node’s input.

3. **Add Function Node to Split Epics**  
   - Type: Function  
   - Name: "Split Epics for Processing"  
   - Code:  
     ```js
     const output = [];
     for (const epic of items) {
       const key = epic.json.key;
       output.push({ json: { epicKey: key } });
     }
     return output;
     ```  
   - Connect "Fetch All Epics from Jira" output to this node.

4. **Add Jira Node to Fetch Epic Details + Linked Issues**  
   - Type: Jira  
   - Name: "Fetch Epic + Linked Issues"  
   - Operation: `get` (retrieve single issue)  
   - Set Issue Key: `={{ $json.epicKey }}` (dynamic from split function)  
   - Configure same Jira credentials  
   - Connect "Split Epics for Processing" output to this node.

5. **Add Function Node to Calculate Health Score**  
   - Type: Function  
   - Name: "Calculate Health Score"  
   - Code:  
     ```js
     const results = [];

     for (const item of items) {
       const issues = item.json.issues || [];
       const total = issues.length || 1;

       let cycleTime = 0;
       let blockers = 0;
       let churn = 0;
       let bugs = 0;

       for (const issue of issues) {
         cycleTime += issue.fields.customfield_cycle_time || 0;
         if (issue.fields.labels && issue.fields.labels.includes('blocker')) blockers++;
         if (issue.fields.status && issue.fields.status.name === 'Reopened') churn++;
         if (issue.fields.issuetype && issue.fields.issuetype.name === 'Bug') bugs++;
       }

       const avgCycleTime = cycleTime / total;
       const bugRatio = bugs / total;
       const churnRatio = churn / total;
       const blockerRatio = blockers / total;

       const healthScore = (avgCycleTime * 0.4) + (bugRatio * 0.3) + (churnRatio * 0.2) + (blockerRatio * 0.1);

       results.push({
         json: {
           epicKey: item.json.epicKey || item.json.key,
           healthScore: Number(healthScore.toFixed(2)),
         }
       });
     }

     return results;
     ```  
   - Connect "Fetch Epic + Linked Issues" output to this node.

6. **Add If Node to Check Health Score Threshold**  
   - Type: If  
   - Name: "Is Score > 0.6 (At Risk)?"  
   - Condition: Number > 0.6 on `{{$json["healthScore"]}}`  
   - Connect "Calculate Health Score" output to this node.

7. **Add Jira Node to Update Epic** (True branch)  
   - Type: Jira  
   - Name: "Update Epic in Jira"  
   - Operation: update  
   - Issue Key: `={{$json["epicKey"]}}`  
   - Update Fields:  
     - Labels: add "At Risk"  
     - Custom Fields: set `customfield_10060` to `{{$json["healthScore"]}}`  
   - Connect True output of If Node to this node.

8. **Add Slack Node to Alert Team** (True branch)  
   - Type: Slack  
   - Name: "Alert Slack Channel"  
   - Channel: `#project-alerts` (or your target channel)  
   - Text:  
     ```
     🚨 *Epic At Risk:* {{$json["epicKey"]}}
     Health Score: {{$json["healthScore"]}}
     https://yourdomain.atlassian.net/browse/{{$json["epicKey"]}}
     ```  
   - Configure Slack API credentials  
   - Connect True output of If Node to this node (parallel to Jira update).

9. **Add Monday.com Node to Update Pulse** (Both branches)  
   - Type: Monday.com  
   - Name: "Update Monday.com Pulse"  
   - Operation: update  
   - Configure Monday.com API credentials  
   - Connect both True and False outputs of If Node to this node.

10. **Add Google Sheets Node to Log Data** (Both branches)  
    - Type: Google Sheets  
    - Name: "Log to Google Sheets"  
    - Operation: append  
    - Sheet ID: Your Google Sheet ID  
    - Range: A:C  
    - Options: valueInputMode = USER_ENTERED  
    - Configure Google API credentials  
    - Connect output of Monday.com node to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                              |
|-----------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Workflow runs every 6 hours to maintain up-to-date health monitoring of Epics.                                               | Scheduling via Cron node                                      |
| Health score weighted formula balances velocity, quality, stability, and dependencies for comprehensive risk assessment.  | See 📈 Health Score sticky note                               |
| Risk threshold at 0.6 score; above this triggers Jira updates and team notifications.                                        | See ⚠️ Risk Check sticky note                                |
| Slack alerts critical for real-time team risk mitigation; ensure Slack credentials and channel are properly configured.     | Slack node and 🚨 Slack Alert sticky note                    |
| Monday.com and Google Sheets integration allows cross-platform synchronization and historical trend tracking.               | See 📋 Monday Pulse and 📊 Sheets Log sticky notes           |
| Custom field ID `customfield_10060` must exist in Jira and be accessible for updates.                                       | Jira update node configuration                               |
| API credentials for Jira, Slack, Monday.com, and Google Sheets must have appropriate permissions and be securely stored.    | Credential management best practices                          |
| Network reliability and API rate limits should be monitored to avoid workflow failures or delays.                           | General operational note                                     |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.