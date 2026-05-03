Weekly HubSpot Lead Report to Slack

https://n8nworkflows.xyz/workflows/weekly-hubspot-lead-report-to-slack-8970


# Weekly HubSpot Lead Report to Slack

### 1. Workflow Overview

This workflow generates and posts a weekly report to a Slack channel summarizing new leads added in HubSpot during the previous week. It is designed primarily to track the volume of new contacts entering the “Lead” stage on a weekly basis. The workflow includes logical blocks for scheduling, data retrieval from HubSpot, filtering by date, summarization, and Slack notification. Additionally, it contains commented-out nodes and notes hinting at an optional extension to report on won deals.

Logical blocks:

- **1.1 Schedule Trigger**: Defines when the workflow runs automatically (weekly).
- **1.2 HubSpot Contacts Retrieval**: Fetches all contacts from HubSpot with relevant lead entry date.
- **1.3 Filtering Leads by Date**: Filters contacts to only those entered as leads during the last week.
- **1.4 Lead Count Summarization**: Calculates the total number of new leads.
- **1.5 Slack Notification**: Sends the summarized lead count as a message to a Slack channel.
- **1.6 Optional Deals Reporting (commented for extension)**: Nodes for fetching and filtering won deals, summing deal values, and notes on usage.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger

- **Overview:**  
  Initiates the workflow on a fixed weekly schedule.

- **Nodes Involved:**  
  - Schedule the report

- **Node Details:**  
  - **Schedule the report**  
    - Type: Schedule Trigger  
    - Configuration: Triggers every week on Monday at 07:00 (7 AM).  
    - Input: None (trigger node)  
    - Output: Connects to “Get all contacts” node  
    - Edge Cases: If n8n instance is offline at trigger time, missed runs may not execute unless catch-up is enabled in n8n settings.  
    - Version: 1.2

---

#### 2.2 HubSpot Contacts Retrieval

- **Overview:**  
  Retrieves all contacts from HubSpot with the property `hs_v2_date_entered_lead` which records when the contact entered the Lead stage.

- **Nodes Involved:**  
  - Get all contacts

- **Node Details:**  
  - **Get all contacts**  
    - Type: HubSpot node (API integration)  
    - Operation: `getAll` contacts, returning all records without pagination limit.  
    - Authentication: OAuth2 via connected HubSpot account.  
    - Properties retrieved: `hs_v2_date_entered_lead` only (for efficiency).  
    - Input: Triggered by “Schedule the report”  
    - Output: Sent to “Filter leads added last week”  
    - Edge Cases:  
      - OAuth token expiration or invalid credentials may cause authentication failure.  
      - Large contact lists may cause timeouts or performance issues.  
      - Missing or null `hs_v2_date_entered_lead` values could cause filtering mismatches.  
    - Version: 2.1

---

#### 2.3 Filtering Leads by Date

- **Overview:**  
  Filters contacts to only those whose `hs_v2_date_entered_lead` property falls within the last 7 days (last week).

- **Nodes Involved:**  
  - Filter leads added last week

- **Node Details:**  
  - **Filter leads added last week**  
    - Type: Filter node  
    - Conditions:  
      - Lead entered date is **before** today (`$today` variable).  
      - Lead entered date is **after** 7 days ago (`$today.minus(7, 'days')`).  
    - Uses expressions converting HubSpot lead entry date string to milliseconds timestamp for comparison.  
    - Input: From “Get all contacts” node  
    - Output: Matching leads forwarded to “Count leads” node  
    - Edge Cases:  
      - Contacts missing `hs_v2_date_entered_lead` will be excluded.  
      - Date parsing errors if property format changes.  
      - Timezone considerations: `$today` uses workflow execution timezone by default; could cause off-by-one day errors if HubSpot dates are in different timezone.  
    - Version: 2.2

---

#### 2.4 Lead Count Summarization

- **Overview:**  
  Counts the number of filtered leads.

- **Nodes Involved:**  
  - Count leads

- **Node Details:**  
  - **Count leads**  
    - Type: Summarize node  
    - Operation: Count distinct `vid` field values (HubSpot contact ID) across filtered leads.  
    - Input: From “Filter leads added last week”  
    - Output: JSON object containing count under key `count_vid` for downstream use.  
    - Edge Cases:  
      - Empty input results in zero count.  
      - Non-unique `vid` values would inflate count (unlikely, as IDs should be unique).  
    - Version: 1.1

---

#### 2.5 Slack Notification

- **Overview:**  
  Sends a message to a configured Slack channel summarizing the total new leads count.

- **Nodes Involved:**  
  - Send report to a Slack channel

- **Node Details:**  
  - **Send report to a Slack channel**  
    - Type: Slack node  
    - Authentication: OAuth2 via connected Slack account  
    - Message Text: `"Last week we generated a total of {{ $json.count_vid }} leads."` — a dynamic expression injecting the counted leads from previous node.  
    - Channel: Selected via OAuth2 Slack account setup (channel ID configured in node, hidden for privacy)  
    - Input: From “Count leads” node  
    - Output: Final endpoint (no further nodes)  
    - Edge Cases:  
      - Slack API rate limits or authentication errors may cause failure.  
      - If `count_vid` is missing or zero, message will still post but may be misleading.  
    - Version: 2.3

---

#### 2.6 Optional Deals Reporting (Commented / For Extension)

- **Overview:**  
  Nodes exist for fetching HubSpot deals, filtering won deals closed in last week, and summing deal values. Currently these nodes are disconnected from the main flow but included with sticky notes suggesting potential use.

- **Nodes Involved:**  
  - Get all deals  
  - Filter won deals last week  
  - Sum deal value  
  - Sticky Note1 ("You can use this for deals too")

- **Node Details:**  
  - **Get all deals**  
    - Type: HubSpot node  
    - Operation: `getAll` deals, retrieving properties `hs_is_closed_won` and `hs_closed_won_date`  
    - Auth: OAuth2 same as contacts  
    - Edge Cases: Large datasets, auth errors  
  - **Filter won deals last week**  
    - Type: Filter node  
    - Conditions:  
      - Deal closed won date is within last 7 days (after `$today.minus(7, 'days')` and before `$today`)  
      - `hs_is_closed_won` equals `true`  
    - Edge Cases: Missing properties, date parsing issues  
  - **Sum deal value**  
    - Type: Summarize node  
    - Operation: Sum of `vid` field (likely a misconfiguration since `vid` is contact ID, not deal value)  
    - Edge Cases: Misconfigured aggregation field, empty inputs  
  - These nodes currently have no downstream connections.

---

### 3. Summary Table

| Node Name                     | Node Type              | Functional Role                   | Input Node(s)             | Output Node(s)                 | Sticky Note                                           |
|-------------------------------|-----------------------|---------------------------------|---------------------------|-------------------------------|-------------------------------------------------------|
| Schedule the report            | Schedule Trigger      | Trigger workflow weekly          | None                      | Get all contacts              | "Optional: adjust when you want the message delivered by editing the day/time on the Schedule Trigger." |
| Get all contacts              | HubSpot API           | Retrieve all contacts            | Schedule the report       | Filter leads added last week  | "Connect your HubSpot account."                        |
| Filter leads added last week  | Filter                | Filter leads entered last week  | Get all contacts          | Count leads                  |                                                       |
| Count leads                  | Summarize             | Count number of filtered leads   | Filter leads added last week | Send report to a Slack channel |                                                       |
| Send report to a Slack channel | Slack                 | Post lead count to Slack channel | Count leads               | None                        | "Connect Slack and choose which channel you want to post to." |
| Get all deals                | HubSpot API           | Retrieve all deals (optional)    | None                      | Filter won deals last week    | "You can use this for deals too"                       |
| Filter won deals last week    | Filter                | Filter deals won last week (opt) | Get all deals             | Sum deal value                |                                                       |
| Sum deal value              | Summarize             | Sum values of deals (optional)   | Filter won deals last week | None                        |                                                       |
| Sticky Note                   | Sticky Note           | Workflow overview note           | None                      | None                        | "Weekly HubSpot “New Leads” report to Slack\n\nThis simple workflow posts a weekly count of contacts that entered the Lead stage last week to a Slack channel." |
| Sticky Note1                  | Sticky Note           | Optional deals note              | None                      | None                        | "You can use this for deals too"                       |
| Sticky Note2                  | Sticky Note           | Empty / placeholder              | None                      | None                        |                                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger every week on Monday at 07:00 (7 AM).  
   - No credentials needed.

2. **Create HubSpot OAuth2 Credential**  
   - Configure OAuth2 credentials for HubSpot API with required scopes to read contacts.

3. **Create “Get all contacts” HubSpot node**  
   - Operation: `getAll` contacts  
   - Return All: true  
   - Authentication: Use HubSpot OAuth2 credentials  
   - Additional Fields:  
     - Properties: `hs_v2_date_entered_lead`  
     - Property Mode: valueOnly (to get direct values)  
   - Connect Schedule Trigger output to this node’s input.

4. **Create “Filter leads added last week” node**  
   - Type: Filter  
   - Add two conditions combined with AND:  
     - Left value: `={{ $json.properties.hs_v2_date_entered_lead.value.toDateTime('ms') }}`  
       Operator: before  
       Right value: `={{ $today }}`  
     - Left value: same as above  
       Operator: after  
       Right value: `={{ $today.minus(7, 'days') }}`  
   - Connect “Get all contacts” output to this node’s input.

5. **Create “Count leads” summarize node**  
   - Type: Summarize  
   - Field to summarize: `vid`  
   - Operation: Count (default aggregation)  
   - Connect “Filter leads added last week” output to this node’s input.

6. **Create Slack OAuth2 Credential**  
   - Configure OAuth2 credentials for Slack with permission to post messages to channels.

7. **Create “Send report to a Slack channel” node**  
   - Type: Slack  
   - Authentication: Slack OAuth2 credentials  
   - Select the channel to post to (set channel ID)  
   - Text: `"Last week we generated a total of {{ $json.count_vid }} leads."`  
   - Connect “Count leads” output to this node’s input.

8. **Connect nodes sequentially:**  
   Schedule Trigger → Get all contacts → Filter leads added last week → Count leads → Send report to a Slack channel

9. *(Optional)* Set up additional nodes for deals reporting if desired:  
   - “Get all deals” HubSpot node with properties `hs_is_closed_won` and `hs_closed_won_date`  
   - Filter deals closed won last week by date and `hs_is_closed_won` = true  
   - Summarize to sum deal values (correct aggregation field should be set based on deal value property)  
   - Configure connections accordingly.

---

### 5. General Notes & Resources

| Note Content                                                                                                    | Context or Link                                                                                     |
|----------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow posts a weekly count of contacts entering Lead stage in HubSpot to Slack channel.                      | Main workflow purpose                                                                               |
| Optional extension exists for deals reporting; users can adapt the workflow similarly.                          | Sticky Note1: “You can use this for deals too”                                                    |
| To adjust delivery time, modify Schedule Trigger node’s settings for day/time.                                  | Sticky Note in setup section                                                                       |
| HubSpot OAuth2 connection must have proper scopes to read contacts and deals.                                  | Credential setup requirement                                                                       |
| Slack OAuth2 connection must have permissions to post messages in selected Slack channel.                       | Credential setup requirement                                                                       |
| Expression parsing uses n8n’s `$today` and date manipulation functions; timezone awareness is important.        | Potential edge case for date filtering                                                             |

---

This documentation fully describes the workflow logic, node configurations, execution flow, and potential edge cases to enable confident reproduction, modification, and troubleshooting.