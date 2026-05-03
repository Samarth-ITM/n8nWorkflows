Generate Dynamic Sales Pipeline Metrics for Slack using Airtable

https://n8nworkflows.xyz/workflows/generate-dynamic-sales-pipeline-metrics-for-slack-using-airtable-6463


# Generate Dynamic Sales Pipeline Metrics for Slack using Airtable

### 1. Workflow Overview

This workflow automates the generation and delivery of a weekly sales pipeline summary report to a Slack channel using data from Airtable. It targets sales managers, founders, and revenue teams who want regular insights into their sales pipeline without manual CRM checks. The report includes key metrics such as open deals count, top deal value, win rate, weighted pipeline value, and total revenue closed.

The workflow is logically divided into these blocks:

- **1.1 Scheduled Trigger**: Initiates the workflow weekly (e.g., Monday at 9 AM).
- **1.2 Data Retrieval from Airtable**: Queries Airtable twice—once to get all open deals (non-won statuses) and once to get all won deals.
- **1.3 Data Merging and Processing**: Merges the two datasets, then uses JavaScript code nodes to calculate and format multiple sales metrics.
- **1.4 Slack Notification**: Sends the compiled, formatted sales report message to a Slack user or channel using OAuth2 authentication.
- **1.5 Documentation and Guidance**: Several sticky notes provide instructions, contextual information, and links for setup and customization.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Trigger

- **Overview:**  
  This block triggers the entire workflow automatically on a weekly basis, specifically every Monday at 9 AM.

- **Nodes Involved:**  
  - Schedule Trigger

- **Node Details:**  
  - **Schedule Trigger**  
    - Type: Schedule Trigger node  
    - Configuration: Runs weekly on Mondays at 9:00 AM  
    - Inputs: None (trigger node)  
    - Outputs: Triggers downstream Airtable search nodes  
    - Edge Cases: If the system clock or timezone is misconfigured, the trigger may run at unexpected times.  
    - Version: 1.2  

- **Sticky Note Related:**  
  - Sticky Note7 explains the schedule trigger purpose and links to Calendly credentials documentation for reference (although Calendly is unrelated here, likely a copy-paste note).

---

#### 1.2 Data Retrieval from Airtable

- **Overview:**  
  This block queries the Airtable base twice to retrieve relevant deal records: one query for all open deals (statuses "Qualified", "Proposal Sent", "Negotiation") and another for all deals marked as "Won".

- **Nodes Involved:**  
  - Search Open Deals  
  - Search Won Deals

- **Node Details:**  
  - **Search Open Deals**  
    - Type: Airtable node  
    - Operation: Search records  
    - Base: Specific Airtable base ID (“appGid2SnQKDjgdhy”)  
    - Table: Deals table (“tblQq0yUYA7dIZxUr”)  
    - Filter: OR formula for statuses “Qualified”, “Proposal Sent”, “Negotiation”  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: List of open deals to Merge Deals node  
    - Edge Cases: Airtable API rate limits, no matching records, or incorrect formula syntax can cause errors.  
    - Note: Uses cached Airtable base and table metadata.  
    - Version: 2.1  

  - **Search Won Deals**  
    - Type: Airtable node  
    - Operation: Search records  
    - Base/Table: Same as above  
    - Filter: Status equals “Won”  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: List of won deals to Merge Deals node  
    - Credentials: Uses Airtable Personal Access Token for authentication  
    - Edge Cases: Authentication errors, empty result sets, API limits  
    - Version: 2.1  

- **Sticky Notes Related:**  
  - Sticky Note3 describes the purpose and details of the “Search Open Deals” node with a link to the Airtable node documentation.  
  - Sticky Note4 similarly explains the “Search Won Deals” node and links to Airtable docs.

---

#### 1.3 Data Merging and Processing

- **Overview:**  
  After retrieving open and won deals, this block merges the two datasets and processes them through two JavaScript code nodes. The first node formats a summary message with basic metrics, and the second node calculates advanced metrics like weighted pipeline and total closed revenue.

- **Nodes Involved:**  
  - Merge Deals  
  - Slack Message Summary (Code node)  
  - Advanced Metrics (Code node)

- **Node Details:**  
  - **Merge Deals**  
    - Type: Merge node  
    - Purpose: Combines outputs from “Search Open Deals” and “Search Won Deals” into a single stream for further processing  
    - Inputs: Two separate inputs from both Airtable search nodes  
    - Outputs: Sends merged items to “Slack Message Summary” node  
    - Edge Cases: If one input is empty, merging still proceeds; node type defaults to "Append" (not explicitly configured here)  
    - Version: 3.2  

  - **Slack Message Summary**  
    - Type: Code node (JavaScript)  
    - Purpose: Splits merged data into open and closed deals, calculates:  
      - Open deals count  
      - Pipeline value (sum of open deals values)  
      - Top deal (highest value open deal)  
      - Win rate (ratio of closed deals to total deals)  
    - Configuration: Custom JS code using array filters, reduce, sorting, and template strings  
    - Inputs: Merged deals from “Merge Deals” node  
    - Outputs: JSON object with computed metrics and formatted summary string  
    - Edge Cases: Handles zero deals to avoid division by zero; if no top deal, uses optional chaining to prevent errors  
    - Version: 2  

  - **Advanced Metrics**  
    - Type: Code node (JavaScript)  
    - Purpose: Computes deeper sales metrics:  
      - Total pipeline value  
      - Weighted pipeline (using stage weights for deal probability)  
      - Total revenue closed (sum of won deals)  
      - Win rate formatted as percentage string  
      - Counts of open and closed deals  
    - Configuration: Uses helper function to safely extract fields; uses a stage-weight mapping object; applies default weight for unknown stages  
    - Inputs: Metrics summary JSON from “Slack Message Summary” node  
    - Outputs: JSON object with formatted metrics for Slack message  
    - Edge Cases: Safely handles missing or malformed numeric values; returns 'N/A' if no deals found  
    - Version: 2  

- **Sticky Note Related:**  
  - Sticky Note1 provides a high-level workflow breakdown across these processing steps.  
  - Sticky Note6 offers links and guidance on the Merge and Code nodes, explaining the calculations and formatting.

---

#### 1.4 Slack Notification

- **Overview:**  
  This block sends the final formatted sales report as a message to Slack using Slack's OAuth2 authenticated API.

- **Nodes Involved:**  
  - Slack Message

- **Node Details:**  
  - **Slack Message**  
    - Type: Slack node  
    - Purpose: Sends a formatted message to a specific Slack user (or channel) with all key metrics from the previous node  
    - Configuration:  
      - Text: Uses mustache expressions to interpolate metrics like total pipeline, weighted pipeline, total closed revenue, win rate, and deal counts  
      - User: Selected from Slack workspace (user ID "U096VCG525P")  
      - Authentication: OAuth2 with Slack credentials  
    - Inputs: Metrics JSON from “Advanced Metrics” node  
    - Outputs: Slack API response (not used further)  
    - Edge Cases: Slack API errors (rate limits, invalid tokens), message formatting errors, network issues  
    - Version: 2.3  

- **Sticky Note Related:**  
  - Sticky Note5 explains the Slack Message node’s function and provides a link to Slack node documentation.

---

#### 1.5 Documentation and Guidance (Sticky Notes)

- **Overview:**  
  Several sticky notes provide contextual information, usage instructions, customization tips, and helpful links for maintainers and users.

- **Nodes Involved:**  
  - Sticky Note (general overview and setup)  
  - Sticky Note1 (workflow breakdown)  
  - Sticky Note3 (Search Open Deals doc)  
  - Sticky Note4 (Search Won Deals doc)  
  - Sticky Note5 (Slack Message doc)  
  - Sticky Note6 (Code and Merge nodes doc)  
  - Sticky Note7 (Schedule Trigger doc)

- **Contents:**  
  - Workflow purpose and audience  
  - Setup instructions (connecting Airtable and Slack, scheduling)  
  - Customization suggestions (changing Slack message format, adding fields, adjusting weight logic)  
  - Links to official n8n documentation and community support (Discord, forum)  

---

### 3. Summary Table

| Node Name            | Node Type           | Functional Role                            | Input Node(s)            | Output Node(s)            | Sticky Note                                                                                             |
|----------------------|---------------------|-------------------------------------------|--------------------------|---------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger      | Schedule Trigger    | Triggers workflow weekly at Monday 9 AM  | —                        | Search Open Deals, Search Won Deals | Sticky Note7: Explains schedule trigger and links to Calendly credentials docs (for reference)           |
| Search Open Deals     | Airtable            | Retrieves open deals from Airtable        | Schedule Trigger          | Merge Deals               | Sticky Note3: Describes open deals query with Airtable node doc link                                   |
| Search Won Deals      | Airtable            | Retrieves won deals from Airtable         | Schedule Trigger          | Merge Deals               | Sticky Note4: Describes won deals query with Airtable node doc link                                    |
| Merge Deals           | Merge               | Merges open and won deals datasets        | Search Open Deals, Search Won Deals | Slack Message Summary     | Sticky Note6: Explains merging and code node processing with links                                    |
| Slack Message Summary | Code                | Calculates basic metrics and formats summary | Merge Deals               | Advanced Metrics          | Sticky Note6                                                                                            |
| Advanced Metrics      | Code                | Calculates advanced metrics and finalizes formatting | Slack Message Summary     | Slack Message             | Sticky Note6                                                                                            |
| Slack Message         | Slack               | Sends the formatted sales report to Slack | Advanced Metrics          | —                         | Sticky Note5: Explains Slack message sending with node doc link                                        |
| Sticky Note           | Sticky Note         | Documentation and setup guidance           | —                        | —                         | Contains full workflow overview, setup, customization, and support info                               |
| Sticky Note1          | Sticky Note         | Workflow breakdown and logic steps         | —                        | —                         | Provides stepwise description of workflow steps                                                      |
| Sticky Note3          | Sticky Note         | Explanation of "Search Open Deals" node    | —                        | —                         | Linked to Airtable node docs                                                                          |
| Sticky Note4          | Sticky Note         | Explanation of "Search Won Deals" node     | —                        | —                         | Linked to Airtable node docs                                                                          |
| Sticky Note5          | Sticky Note         | Explanation of Slack Message node          | —                        | —                         | Linked to Slack node docs                                                                             |
| Sticky Note6          | Sticky Note         | Explanation of Merge and Code nodes         | —                        | —                         | Links and info about code node logic and merge node                                                  |
| Sticky Note7          | Sticky Note         | Explanation of Schedule Trigger node       | —                        | —                         | Links to schedule trigger and Calendly credentials docs                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to run weekly on Mondays at 9:00 AM  
   - No input credentials needed  

2. **Create Airtable node “Search Open Deals”:**  
   - Operation: Search  
   - Airtable Base: Select or enter your base ID (e.g., “appGid2SnQKDjgdhy”)  
   - Table: Deals table (e.g., “tblQq0yUYA7dIZxUr”)  
   - Filter formula: `OR({Status} = "Qualified", {Status} = "Proposal Sent", {Status} = "Negotiation")`  
   - Connect credentials for Airtable Personal Access Token  
   - Connect input from Schedule Trigger  

3. **Create Airtable node “Search Won Deals”:**  
   - Same base and table as above  
   - Filter formula: `{Status} = "Won"`  
   - Use the same Airtable credentials  
   - Connect input from Schedule Trigger  

4. **Create a Merge node “Merge Deals”:**  
   - Default mode (Append)  
   - Connect “Search Open Deals” to input 0  
   - Connect “Search Won Deals” to input 1  

5. **Create a Code node “Slack Message Summary”:**  
   - Language: JavaScript  
   - Paste the provided script that:  
     - Separates open and won deals  
     - Calculates open deals count, pipeline value, top deal, and win rate  
     - Formats a summary string  
   - Connect input from “Merge Deals” node  

6. **Create a second Code node “Advanced Metrics”:**  
   - Language: JavaScript  
   - Paste the provided script that:  
     - Calculates total pipeline, weighted pipeline, total closed revenue, and win rate  
     - Uses stage weights for weighting pipeline value  
     - Outputs a JSON object with formatted metric strings and counts  
   - Connect input from “Slack Message Summary” node  

7. **Create a Slack node “Slack Message”:**  
   - Operation: Send Message  
   - Authentication: OAuth2 with Slack credentials (set up Slack app with chat:write scope)  
   - User: Select the user or channel to send the message to (e.g., user ID “U096VCG525P”)  
   - Message text: Use the following template with mustache expressions to insert metrics:  
     ```
     📊 *Weekly Sales Report*  
     • 🧮 *Total Pipeline:* {{ $json.totalPipeline }}  
     • ⚖️ *Weighted Pipeline:* {{ $json.weightedPipeline }}  
     • 🏆 *Total Closed (All Time):* {{ $json.totalClosed }}  
     • 📈 *Win Rate:* {{ $json.winRate }}  
     • 🔄 *Open Deals:* {{ $json.openDealsCount }}  
     • ✅ *Closed Deals:* {{ $json.closedDealsCount }}  
     _This report was generated automatically using n8n._
     ```  
   - Connect input from “Advanced Metrics” node  

8. **Connect all nodes accordingly:**  
   - Schedule Trigger → Search Open Deals and Search Won Deals  
   - Search Open Deals & Search Won Deals → Merge Deals  
   - Merge Deals → Slack Message Summary  
   - Slack Message Summary → Advanced Metrics  
   - Advanced Metrics → Slack Message  

9. **Test the workflow:**  
   - Run manually or wait for scheduled trigger  
   - Confirm correct data retrieval from Airtable  
   - Verify calculation correctness and Slack message formatting  
   - Adjust stage weights or Slack message formatting as needed  

---

### 5. General Notes & Resources

| Note Content                                                                                                              | Context or Link                                              |
|---------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| This workflow sends weekly pipeline summaries with key sales metrics to Slack for sales managers and revenue teams.       | Workflow purpose from Sticky Note                             |
| Airtable node documentation for reference when setting up data queries.                                                   | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.airtable/ |
| Slack node documentation for message sending and authentication setup guidance.                                            | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.slack/   |
| Join the n8n Discord or Forum for community support and help.                                                             | Discord: https://discord.com/invite/XPKeKXeB7d                    |
| Customize the weighting logic in the Code node to fit your sales pipeline stages or probabilities.                        | Sticky Note mentions customization tips                       |
| This workflow requires an Airtable base with a Deals table containing at least “Status” and “Value” fields.               | Requirement from Sticky Note                                   |
| Slack OAuth2 credentials require a Slack app with chat:write permission and OAuth tokens configured in n8n.               | Credential setup best practice                                |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated n8n workflow export. This processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.