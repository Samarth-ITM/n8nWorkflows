Daily B2B Lead Monitoring: Website Visitor Digest with ProspectPro and Gmail

https://n8nworkflows.xyz/workflows/daily-b2b-lead-monitoring--website-visitor-digest-with-prospectpro-and-gmail-8689


# Daily B2B Lead Monitoring: Website Visitor Digest with ProspectPro and Gmail

### 1. Workflow Overview

This workflow automates daily monitoring and notification of B2B website visitors (prospects) using ProspectPro data and sends a digest email via Gmail. It targets businesses that want a daily summary of companies visiting their website, highlighting qualified prospects and recent activity.

**Logical Blocks:**

- **1.1 Input Trigger and Time Setup:** Schedule trigger and time calculation to define the 24-hour filter window.
- **1.2 Prospect Retrieval:** Query ProspectPro API for prospects modified in the last 24 hours.
- **1.3 Prospect Filtering:** Filter out unqualified prospects and those without recent visits.
- **1.4 Result Check:** Conditional branch to continue or cancel workflow based on prospect availability.
- **1.5 Prospect List Composition:** Format a human-readable summary of prospects for the email body.
- **1.6 Notification Sending:** Send the formatted prospect summary by Gmail.
- **1.7 Error and Completion Handling:** Manage errors and mark workflow end steps.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Trigger and Time Setup

- **Overview:**  
  This block triggers the workflow daily at 07:50 and calculates the timestamp 24 hours before the current run time, rounded down to the closest hour, to filter prospect data.

- **Nodes Involved:**  
  - Daily, 07:50 (Schedule Trigger)  
  - Set Filter Time (Code)

- **Node Details:**

  - **Daily, 07:50**  
    - Type: Schedule Trigger  
    - Configuration: Executes once daily at 07:50 AM.  
    - Inputs: None (trigger node)  
    - Outputs: Triggers the next node `Set Filter Time`.  
    - Edge cases: None significant; ensure server time zone matches expected schedule.

  - **Set Filter Time**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Receives timestamp from trigger event.  
      - Subtracts 1 day from the timestamp.  
      - Rounds down to the hour for consistency in API filter.  
      - Outputs two fields: original timestamp and calculated `minusOneDayRoundedHour`.  
    - Key expressions: Uses `$json["timestamp"]` and native JS `Date` methods.  
    - Inputs: Timestamp JSON from trigger.  
    - Outputs: Timestamp object for filtering in next node.  
    - Edge cases: Invalid or missing timestamp input could cause runtime error; ensure trigger provides timestamp.

---

#### 1.2 Prospect Retrieval

- **Overview:**  
  Uses the ProspectPro API node to fetch prospects modified since the calculated filter time, including detailed company information.

- **Nodes Involved:**  
  - Get Prospects (ProspectPro node)

- **Node Details:**

  - **Get Prospects**  
    - Type: ProspectPro API node (custom n8n node)  
    - Configuration:  
      - Fetch companies with `changed_time` greater than or equal to `minusOneDayRoundedHour`.  
      - Filters for companies with at least some visits (`total_visits: 666` is a placeholder, likely ignored or a fixed filter).  
      - Sorts results by changed time descending.  
      - Requests detailed company info.  
    - Credentials: Uses ProspectPro API credentials (OAuth or API key).  
    - Inputs: Output from `Set Filter Time` node for `from_changed_time` filter.  
    - Outputs: JSON with list of companies under `companies` property.  
    - On error: Workflow continues but routes error output to `Error: ProspectPro` node.  
    - Edge cases: API rate limits, invalid credentials, network timeouts, empty results.

---

#### 1.3 Prospect Filtering

- **Overview:**  
  Filters out prospects that are disqualified (`label === 0`) or have not visited since the cutoff time, producing a list of qualified recent visitors.

- **Nodes Involved:**  
  - Select Prospects (Code)

- **Node Details:**

  - **Select Prospects**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Extracts cutoff timestamp from `Set Filter Time`.  
      - Filters `companies` array where:  
        - `last_visit` timestamp is after cutoff (converted to seconds).  
        - `label` is not zero (0 = disqualified).  
      - Outputs total count and filtered prospects array.  
    - Inputs: JSON with `companies` from `Get Prospects` and cutoff time from `Set Filter Time`.  
    - Outputs: `{ total: number, prospects: array }` for downstream use.  
    - Edge cases: Missing or malformed `companies` array, timestamp conversion errors.

---

#### 1.4 Result Check

- **Overview:**  
  Checks if there are any prospects after filtering; if none, stops the workflow to avoid sending empty notifications.

- **Nodes Involved:**  
  - Has Prospects? (If)  
  - No Prospects, No Email (NoOp)

- **Node Details:**

  - **Has Prospects?**  
    - Type: If  
    - Configuration:  
      - Condition: Check if `.total` (from `Select Prospects`) > 0.  
      - If true: continue to `Create a Prospect List`.  
      - If false: go to `No Prospects, No Email`.  
    - Inputs: Result of `Select Prospects`.  
    - Outputs: Two branches - true or false.  
    - Edge cases: Non-numeric or missing `.total` could cause logic failure.

  - **No Prospects, No Email**  
    - Type: NoOp (Stops workflow)  
    - Configuration: No operation; marks end of workflow when no prospects found.  
    - Inputs: False branch from `Has Prospects?`.  
    - Edge cases: None.

---

#### 1.5 Prospect List Composition

- **Overview:**  
  Formats up to 20 prospects into an HTML message string, prioritizing qualified prospects (`label === 1`), and including key details like name, city, tags, visits, and pageviews.

- **Nodes Involved:**  
  - Create a Prospect List (Code)

- **Node Details:**

  - **Create a Prospect List**  
    - Type: Code (JavaScript)  
    - Configuration:  
      - Splits prospects into qualified (`label === 1`) and others.  
      - Concatenates qualified first, then others.  
      - Limits to top 20 prospects.  
      - Builds HTML lines including:  
        - Company name in bold  
        - Label (qualified or city/country)  
        - Domain as clickable link  
        - Tags as comma-separated list  
        - Number of visits and pageviews in italic  
      - Joins lines with `<br>` to produce message string.  
    - Inputs: JSON with `prospects` from `Has Prospects?`.  
    - Outputs: Single `{ message: string }` with formatted HTML.  
    - Edge cases: Missing fields (city, domain, tags) handled gracefully. Empty prospect list results in empty message.

---

#### 1.6 Notification Sending

- **Overview:**  
  Sends an email via Gmail containing the daily prospect summary, with subject and body dynamically set based on prospect count and message content.

- **Nodes Involved:**  
  - Send Notification (Gmail)  
  - Done (NoOp)

- **Node Details:**

  - **Send Notification**  
    - Type: Gmail node (OAuth2)  
    - Configuration:  
      - Subject: Includes total prospect count dynamically.  
      - Message body: HTML formatted including greetings, prospect count, up to 20 prospects summary, and link to ProspectPro portal.  
      - Options: Attribution disabled.  
    - Credentials: Gmail OAuth2 credentials required and configured.  
    - Inputs: Formatted message from `Create a Prospect List` and total count from `Has Prospects?`.  
    - On error: Continues on error, routes to `Error: Gmail`.  
    - Outputs: Success routes to `Done`.  
    - Edge cases: Gmail API rate limits, auth token expiration, email sending failures.

  - **Done**  
    - Type: NoOp  
    - Configuration: Marks workflow completion after email send.  
    - Inputs: Success output from `Send Notification`.  
    - Edge cases: None.

---

#### 1.7 Error and Completion Handling

- **Overview:**  
  Handles errors from main API interactions and marks workflow end states.

- **Nodes Involved:**  
  - Error: ProspectPro (NoOp)  
  - Error: Gmail (NoOp)  
  - Sticky Notes (Documentation and guidance)

- **Node Details:**

  - **Error: ProspectPro**  
    - Type: NoOp  
    - Configuration: Placeholder to handle ProspectPro API errors gracefully.  
    - Inputs: Error branch from `Get Prospects`.  
    - Edge cases: Could be extended to log errors or notify via other channels.

  - **Error: Gmail**  
    - Type: NoOp  
    - Configuration: Placeholder to handle Gmail send errors gracefully.  
    - Inputs: Error branch from `Send Notification`.  
    - Edge cases: Same as above.

  - **Sticky Notes**  
    - Provide documentation for error handling best practices, workflow purpose, node explanations, and usage instructions.

---

### 3. Summary Table

| Node Name            | Node Type                  | Functional Role                         | Input Node(s)       | Output Node(s)           | Sticky Note                                                                                          |
|----------------------|----------------------------|---------------------------------------|---------------------|--------------------------|----------------------------------------------------------------------------------------------------|
| Daily, 07:50         | Schedule Trigger           | Triggers workflow daily at 07:50      | —                   | Set Filter Time           | ## Trigger: Daily Workflow executes daily at 07:50.                                                |
| Set Filter Time       | Code                      | Calculates 24h filter timestamp       | Daily, 07:50        | Get Prospects             | ## Set Filter Time Simple code for daily notifications. Adjust when changing your execution interval. |
| Get Prospects         | ProspectPro API Node      | Retrieves prospects changed last 24h  | Set Filter Time     | Select Prospects, Error: ProspectPro | ## Get Prospects Get all prospects modified in last 24 hours.                                      |
| Select Prospects      | Code                      | Filters prospects by visit & qualification | Get Prospects       | Has Prospects?            | ## Filter Prospects Remove disqualified prospects and prospects without recent visits.             |
| Has Prospects?        | If                        | Checks if there are prospects to report | Select Prospects    | Create a Prospect List, No Prospects, No Email | ## Check Results Check if there are any prospects to report.                                       |
| No Prospects, No Email| NoOp                      | Stops workflow if no prospects found  | Has Prospects? (false) | —                        | ## No Results Cancel workflow when there isn't any activity to report.                             |
| Create a Prospect List| Code                      | Formats prospect list for email        | Has Prospects? (true)| Send Notification         | ## Prospect List Define prospect data to include in notification.                                 |
| Send Notification     | Gmail                     | Sends daily notification email         | Create a Prospect List | Done, Error: Gmail        | ## Notification Configure and compose your notification.                                          |
| Done                 | NoOp                      | Marks successful workflow completion   | Send Notification   | —                        | ## Workflow Completed Optionally add following steps, like logging.                               |
| Error: ProspectPro    | NoOp                      | Handles ProspectPro API errors          | Get Prospects (error) | —                        | ## Error handling Please always handle potential errors properly.                                  |
| Error: Gmail          | NoOp                      | Handles Gmail sending errors            | Send Notification (error) | —                      | ## Error handling Please always handle potential errors properly.                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to run daily at 07:50.

2. **Create Code Node “Set Filter Time”**  
   - Type: Code (JavaScript)  
   - Configure to receive timestamp from trigger.  
   - JavaScript code:  
     ```javascript
     const ts = new Date($json["timestamp"]);
     ts.setDate(ts.getDate() - 1);
     ts.setMinutes(0, 0, 0);
     return {
       original: $json["timestamp"],
       minusOneDayRoundedHour: ts.toISOString(),
     };
     ```
   - Connect output of Schedule Trigger to this node.

3. **Create ProspectPro Node “Get Prospects”**  
   - Type: ProspectPro API Node (requires installation of `@bedrijfsdatanl/n8n-nodes-prospectpro.prospectpro`)  
   - Configure:  
     - details: true  
     - filterOptions: `from_changed_time` set to `={{ $json.minusOneDayRoundedHour }}`  
     - sortingOptions: sort by `changed_time` descending  
     - paginationOptions: default (no pagination)  
   - Set credentials with ProspectPro API credentials.  
   - Connect output of “Set Filter Time” to this node.

4. **Create Code Node “Select Prospects”**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const cutoff = new Date($('Set Filter Time').item.json.minusOneDayRoundedHour).getTime() / 1000;
     const prospects = $json["companies"] || [];
     const visited = prospects.filter(c => c.last_visit >= cutoff && c.label !== 0);
     return {
       total: visited.length,
       prospects: visited
     };
     ```
   - Connect output of “Get Prospects” (main output) to this node.  
   - Connect error output of “Get Prospects” to an error handling node (optional).

5. **Create If Node “Has Prospects?”**  
   - Type: If  
   - Condition: `$json.total` > 0  
   - Connect output of “Select Prospects” to this node.

6. **Create NoOp Node “No Prospects, No Email”**  
   - Type: NoOp  
   - Connect false branch of “Has Prospects?” to this node.

7. **Create Code Node “Create a Prospect List”**  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const prospects = $json.prospects || [];
     const hot = prospects.filter(p => p.label === 1);
     const rest = prospects.filter(p => p.label !== 1);
     const sortedProspects = hot.concat(rest);
     const limitedProspects = sortedProspects.slice(0, 20);
     const lines = [];
     if (limitedProspects.length) {
       limitedProspects.forEach(p => {
         const name = `<b>${p.name}</b>`;
         const label = p.label === 1 ? ' (Gekwalificeerd)' : p.city ? `(${p.city}${p.country_code && p.country_code !== 'NL' ? `, ${p.country_code}` : ''})` : '';
         const url = p.url && p.domain ? `<a href="${p.url}" target="_blank">${p.domain}</a>` : p.domain ? `<a href="${p.domain}" target="_blank">${p.domain}</a>` : '';
         const tags = p.tags ? p.tags.split('|').join(", ") : '';
         lines.push(name + (label ? ' ' + label : ''));
         lines.push(url + (url && tags ? ' - ' : '') + tags);
         lines.push(`<i>Aantal bezoeken: ${p.total_visits}. Bezochte pagina's: ${p.pageviews}</i>`);
         lines.push("");
       });
     }
     return { message: lines.join("<br>") };
     ```
   - Connect true branch of “Has Prospects?” to this node.

8. **Create Gmail Node “Send Notification”**  
   - Type: Gmail (OAuth2)  
   - Subject:  
     ```
     =ProspectPro: {{ $('Has Prospects?').item.json.total }} prospects hebben je website bezocht
     ```  
   - Message (HTML):  
     ```
     =Goedemorgen!<br><br>In de afgelopen 24 uur hebben <b>{{ $('Has Prospects?').item.json.total }} prospects</b> jouw website bezocht. {{ $('Has Prospects?').item.json.total > 20 ? "Dit zijn de 20 belangrijkste." : '' }}<br><br>
     {{ $json.message }}
     <br><br><a href="https://mijn.prospectpro.nl" target="_blank">Bekijk alle prospects in ProspectPro.</a>
     ```  
   - Disable attribution.  
   - Set Gmail OAuth2 credentials.  
   - Connect output of “Create a Prospect List” to this node.

9. **Create NoOp Node “Done”**  
   - Type: NoOp  
   - Connect success output of “Send Notification” to this node.

10. **Create NoOp Nodes “Error: ProspectPro” and “Error: Gmail”**  
    - Type: NoOp  
    - Connect error outputs of “Get Prospects” and “Send Notification” respectively to these nodes.  
    - Optionally extend these nodes to log errors or send alerts.

11. **Add Sticky Notes (Optional)**  
    - Add documentation sticky notes to explain each block, error handling, and best practices as per your organizational standards.

---

### 5. General Notes & Resources

| Note Content                                                                                                                              | Context or Link                                        |
|-------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------|
| This workflow collects all prospects that have visited your website in the last 24 hours and sends a daily email notification.            | ProspectPro portal: https://mijn.prospectpro.nl       |
| ProspectPro API documentation is available for customizing filters and sorting: https://docs.prospectpro.nl                             | API Docs                                              |
| Example notification screenshot available at:                                                                                             | https://www.bedrijfsdata.nl/wp-content/uploads/2025/09/Schermafbeelding-2025-09-17-om-16.52.13.png#full-width |
| Error handling best practice: log errors in Google Sheet or notify teams via Slack/Telegram/email to ensure visibility and troubleshooting | See Sticky Note7 content for details                   |
| Adjust `Set Filter Time` node if changing schedule interval to maintain correct filtering window.                                          | See Sticky Note11                                      |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow built with n8n, adhering strictly to current content policies without any illegal, offensive, or protected elements. All handled data is legal and public.