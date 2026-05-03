Monitor German Company Register & Financial Events with Implisense API & Notifications

https://n8nworkflows.xyz/workflows/monitor-german-company-register---financial-events-with-implisense-api---notifications-11749


# Monitor German Company Register & Financial Events with Implisense API & Notifications

### 1. Workflow Overview

This workflow automates the monitoring of German companies' register, financial, and news-related events by leveraging the Implisense API (also known as German Company Data API). It is designed to process a list of companies (e.g., from CRM or other lead sources), look up these companies in the German company register, fetch recent events relevant to each, normalize and filter those events based on categories of interest, deduplicate them, and prepare notifications with urgency levels. The workflow is structured to handle batch processing, rate limiting, error handling, and result summarization. It can be extended to send notifications via email, chat, webhooks, or integrate with downstream systems such as CRMs or data stores.

**Logical blocks:**

- **1.1 Initialization & Input Reception:** Setting up credentials, receiving or mocking input company data.
- **1.2 Batch Processing & Rate Limiting:** Splitting input data into batches and pacing requests to respect API limits.
- **1.3 Company Lookup & Event Retrieval:** Resolving companies via the Implisense API and fetching their event data.
- **1.4 Event Normalization & Error Handling:** Parsing API responses to uniform event objects and managing API errors.
- **1.5 Filtering & Deduplication:** Selecting only relevant event categories and removing duplicate events.
- **1.6 Notification Preparation & Dispatch:** Formatting notification messages based on event urgency and sending them.
- **1.7 Logging and Summary:** Logging successes/failures and compiling an execution summary.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization & Input Reception

- **Overview:**  
  This block initializes the workflow context, setting API credentials and workflow run identifiers. It also provides the input list of companies to monitor, currently mocked for testing purposes.

- **Nodes Involved:**  
  - Config  
  - When clicking ‘Execute workflow’ (manual trigger)  
  - Mock Lead Input

- **Node Details:**

  - **Config**  
    - Type: Set  
    - Role: Stores configuration variables such as the RapidAPI key (`x-rapidapi-key`) and the current workflow execution ID (`workflow_run_id`).  
    - Key expressions: The workflow execution ID is dynamically set from `$execution.id`.  
    - Inputs: Manual trigger node.  
    - Outputs: Mock Lead Input.

    - Failure cases: Missing or incorrect API key will cause API calls downstream to fail.

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual workflow execution.  
    - Inputs: User trigger.  
    - Outputs: Config node.  
    - No failure modes, manual start only.

  - **Mock Lead Input**  
    - Type: Function  
    - Role: Provides mock company lead data for testing; can be replaced with real data sources.  
    - Configuration: Returns one example company JSON with name, domain, and location.  
    - Inputs: Config node.  
    - Outputs: Split Batches node.  
    - Edge cases: Needs replacement for real use cases; otherwise processes a static single company.

---

#### 1.2 Batch Processing & Rate Limiting

- **Overview:**  
  This block divides the input leads into manageable batches and applies wait times between API calls to respect rate limits enforced by the German Company Data API.

- **Nodes Involved:**  
  - Split Batches  
  - Rate Limit

- **Node Details:**

  - **Split Batches**  
    - Type: SplitInBatches  
    - Role: Splits the company lead input into batches of 5 to reduce load on the API and enable throttling.  
    - Configuration: Batch size set to 5.  
    - Inputs: Mock Lead Input.  
    - Outputs: Create Summary (after all batches processed) and Rate Limit (for sequential batch processing).  
    - Edge cases: Batch size too large may cause API throttling; too small may slow processing.

  - **Rate Limit**  
    - Type: Wait  
    - Role: Delays execution by 2 seconds between batches to avoid hitting API rate limits.  
    - Configuration: Wait amount set to 2 seconds.  
    - Inputs: Split Batches (for each batch except first).  
    - Outputs: Lookup Company node (start company lookup for batch).  
    - Edge cases: Insufficient delay may cause rate limit errors; excessive delay slows workflow.

---

#### 1.3 Company Lookup & Event Retrieval

- **Overview:**  
  For each company in a batch, look up detailed company data via the Implisense API and then fetch recent events associated with that company.

- **Nodes Involved:**  
  - Lookup Company  
  - Get Events  
  - API Success?

- **Node Details:**

  - **Lookup Company**  
    - Type: HTTP Request  
    - Role: Queries the Implisense API to resolve company details based on company name and domain.  
    - Configuration: POST request to `https://german-company-data.p.rapidapi.com/lookup` with body parameters including company name and domain, limit to 10 results, and necessary RapidAPI headers with API key from Config node.  
    - Inputs: Rate Limit node.  
    - Outputs: Get Events node.  
    - Edge cases: API authentication failure, no matching companies, malformed input.

  - **Get Events**  
    - Type: HTTP Request  
    - Role: Fetches recent events for the first matched company from Lookup Company.  
    - Configuration: GET request to `https://german-company-data.p.rapidapi.com/companies/{{company_id}}/events` with query parameters filtering categories and date range since 2017, limited to 100 events, headers with API key.  
    - Inputs: Lookup Company.  
    - Outputs: API Success? node.  
    - Edge cases: API failures, empty event list, invalid company ID.

  - **API Success?**  
    - Type: If  
    - Role: Checks if the API response status code is 200 (success).  
    - Configuration: Condition checks `$json.statusCode === 200`.  
    - Inputs: Get Events.  
    - Outputs: Normalize Events (on success) or Log API Error (on failure).  
    - Edge cases: Non-200 status codes, missing statusCode field.

---

#### 1.4 Event Normalization & Error Handling

- **Overview:**  
  This block converts raw event data into a standardized format, extracting key event properties, and logs errors if the API response failed.

- **Nodes Involved:**  
  - Normalize Events  
  - Log API Error  
  - Merge Results

- **Node Details:**

  - **Normalize Events**  
    - Type: Code  
    - Role: Processes the event array from API response, producing uniform event objects with consistent fields such as event type, category code, title, text, timestamp, and source. Also attaches company and workflow run metadata.  
    - Inputs: API Success? (success branch).  
    - Outputs: Merge Results node.  
    - Edge cases: Empty event list returns empty output; missing fields defaulted (e.g., unknown type).

  - **Log API Error**  
    - Type: Code  
    - Role: Logs API failure details including error message and status code, returning error JSON for downstream handling.  
    - Inputs: API Success? (failure branch).  
    - Outputs: Merge Results node.  
    - Edge cases: Missing error message fields; logs to console only.

  - **Merge Results**  
    - Type: Merge  
    - Role: Combines successful normalized events and error logs into a single stream for further processing.  
    - Configuration: Combine mode, including unpaired items, merged by position.  
    - Inputs: Normalize Events and Log API Error.  
    - Outputs: Filter Relevant node.

---

#### 1.5 Filtering & Deduplication

- **Overview:**  
  This block filters events to only those in relevant categories and removes duplicates to avoid repeated notifications.

- **Nodes Involved:**  
  - Filter Relevant  
  - Deduplicate

- **Node Details:**

  - **Filter Relevant**  
    - Type: If  
    - Role: Passes events only if their `event_category` is not empty and matches a regex for categories of interest such as MANAGEMENT_AND_TEAM, FINANCES_AND_CAPITAL, MERGERS_AND_ACQUISITIONS, BANKRUPTCY, INSOLVENCY, EXPANSION, RELOCATION.  
    - Inputs: Merge Results.  
    - Outputs: Deduplicate node (true branch).  
    - Edge cases: Events missing category or with unmatched categories are filtered out.

  - **Deduplicate**  
    - Type: RemoveDuplicates  
    - Role: Removes duplicate events to prevent redundant notifications.  
    - Inputs: Filter Relevant (true).  
    - Outputs: Prepare Notification node.  
    - Edge cases: No deduplication criteria explicitly shown; defaults to entire JSON object comparison.

---

#### 1.6 Notification Preparation & Dispatch

- **Overview:**  
  Prepares formatted notification messages with urgency levels based on event categories and dispatches them to the configured notification channel.

- **Nodes Involved:**  
  - Prepare Notification  
  - Email, Chat, Webhook etc.  
  - Notification Sent? (disabled)  
  - Log Success  
  - Log Failed  
  - Merge Notifications  
  - Loop Continue

- **Node Details:**

  - **Prepare Notification**  
    - Type: Code  
    - Role: Constructs a formatted message for each event including urgency (HIGH, MEDIUM, LOW) determined by event category, event age in hours, and key event details.  
    - Inputs: Deduplicate.  
    - Outputs: Email, Chat, Webhook etc.  
    - Edge cases: Missing event fields default to placeholders; urgency defaults to LOW if category unrecognized.

  - **Email, Chat, Webhook etc.**  
    - Type: NoOp (placeholder)  
    - Role: Placeholder node where actual notification sending nodes should be inserted (e.g., email node, Slack node).  
    - Inputs: Prepare Notification.  
    - Outputs: Notification Sent? node.  
    - Edge cases: Needs user to add real notification nodes.

  - **Notification Sent?**  
    - Type: If (disabled)  
    - Role: Intended to check for error in sending notification; currently disabled.  
    - Inputs: Email, Chat, Webhook etc.  
    - Outputs: Log Success (true), Log Failed (false).  
    - Edge cases: Disabled, so no effect currently.

  - **Log Success**  
    - Type: Code  
    - Role: Logs successful notification sending to console and marks status as 'sent'.  
    - Inputs: Notification Sent? (true branch).  
    - Outputs: Merge Notifications.  
    - Edge cases: Only logs to console.

  - **Log Failed**  
    - Type: Code  
    - Role: Logs failed notification attempts, marks status as 'failed'.  
    - Inputs: Notification Sent? (false branch).  
    - Outputs: Merge Notifications.  
    - Edge cases: Only logs to console.

  - **Merge Notifications**  
    - Type: Merge  
    - Role: Combines success and failure logs before continuing the loop.  
    - Inputs: Log Success and Log Failed.  
    - Outputs: Loop Continue.  
    - Edge cases: None.

  - **Loop Continue**  
    - Type: NoOp  
    - Role: Acts as a loop control node to trigger next batch processing.  
    - Inputs: Merge Notifications.  
    - Outputs: Split Batches (starting next batch).  
    - Edge cases: None.

---

#### 1.7 Logging and Summary

- **Overview:**  
  After all batches are processed, this block compiles a summary of the workflow execution, counting total events processed, notifications sent, failed, and success rate.

- **Nodes Involved:**  
  - Create Summary

- **Node Details:**

  - **Create Summary**  
    - Type: Code  
    - Role: Aggregates all processed events and calculates statistics such as total events, sent/failed counts, success rate, and timestamp.  
    - Inputs: Split Batches (final output after all batches processed).  
    - Outputs: Terminal node (can be connected to further output or storage).  
    - Edge cases: Handles zero events gracefully by setting success rate to 0.

---

### 3. Summary Table

| Node Name               | Node Type             | Functional Role                          | Input Node(s)               | Output Node(s)               | Sticky Note                                                  |
|-------------------------|-----------------------|----------------------------------------|-----------------------------|-----------------------------|--------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger         | Workflow entry point                    |                             | Config                      |                                                              |
| Config                  | Set                   | Stores API key and workflow run ID     | When clicking ‘Execute workflow’ | Mock Lead Input             |                                                              |
| Mock Lead Input         | Function              | Provides mock company lead data         | Config                      | Split Batches               | ## Init                                                      |
| Split Batches           | SplitInBatches        | Splits leads into batches                | Mock Lead Input, Loop Continue | Create Summary, Rate Limit | ## Init                                                      |
| Rate Limit              | Wait                  | Waits 2 seconds between batches          | Split Batches               | Lookup Company              | ## Lookup & Data                                            |
| Lookup Company          | HTTP Request          | Resolves company details via API         | Rate Limit                  | Get Events                  | ## Lookup & Data                                            |
| Get Events              | HTTP Request          | Fetches recent events for company        | Lookup Company              | API Success?                | ## Lookup & Data                                            |
| API Success?            | If                    | Checks for successful API response       | Get Events                  | Normalize Events, Log API Error | ## Lookup & Data                                        |
| Normalize Events        | Code                  | Normalizes events into uniform format    | API Success? (success)      | Merge Results               | ## Lookup & Data                                            |
| Log API Error           | Code                  | Logs API error details                    | API Success? (failure)      | Merge Results               | ## Lookup & Data                                            |
| Merge Results           | Merge                 | Combines normalized events and errors    | Normalize Events, Log API Error | Filter Relevant           | ## Lookup & Data                                            |
| Filter Relevant         | If                    | Filters events by relevant categories     | Merge Results               | Deduplicate                 | ## Lookup & Data                                            |
| Deduplicate             | RemoveDuplicates      | Removes duplicate events                   | Filter Relevant             | Prepare Notification        | ## Lookup & Data                                            |
| Prepare Notification    | Code                  | Formats notification message with urgency | Deduplicate                 | Email, Chat, Webhook etc.   | ## Optional Send Notification                             |
| Email, Chat, Webhook etc.| NoOp                   | Placeholder for actual notification senders | Prepare Notification      | Notification Sent?          | ## Optional Send Notification                             |
| Notification Sent?      | If (disabled)          | Checks if notification was sent           | Email, Chat, Webhook etc.   | Log Success, Log Failed     | ## Optional Send Notification                             |
| Log Success             | Code                  | Logs successful notification send          | Notification Sent? (true)   | Merge Notifications         | ## Optional Send Notification                             |
| Log Failed              | Code                  | Logs failed notification send              | Notification Sent? (false)  | Merge Notifications         | ## Optional Send Notification                             |
| Merge Notifications     | Merge                 | Combines success and failure logs          | Log Success, Log Failed     | Loop Continue               | ## Optional Send Notification                             |
| Loop Continue           | NoOp                  | Controls loop to proceed with next batch   | Merge Notifications         | Split Batches               | ## Optional Send Notification                             |
| Create Summary          | Code                  | Aggregates counts and success rates        | Split Batches               |                             |                                                              |
| 🏗️ Architecture Notes    | Sticky Note            | Workflow purpose and setup instructions   |                             |                             | # B2B Register Event Monitoring for German Companies...    |
| Sticky Note1            | Sticky Note            | Init block label                           |                             |                             | ## Init                                                      |
| Sticky Note2            | Sticky Note            | Lookup & Data block label                  |                             |                             | ## Lookup & Data                                            |
| Sticky Note3            | Sticky Note            | Optional notification sending block label |                             |                             | ## Optional Send Notification                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: `When clicking ‘Execute workflow’`  
   - Type: Manual Trigger  
   - Purpose: Start the workflow manually.

2. **Create Set Node for Config**  
   - Name: `Config`  
   - Type: Set  
   - Parameters:  
     - Add string parameter `x-rapidapi-key` with your RapidAPI key (replace `"XXXX"`).  
     - Add string parameter `workflow_run_id` with expression: `{{$execution.id}}`.  
   - Connect from Manual Trigger.

3. **Create Function Node for Mock Input**  
   - Name: `Mock Lead Input`  
   - Type: Function  
   - Code:  
     ```javascript
     return [
       {
         json: {
           leadId: "L-12345",
           companyName: "Implisense",
           domain: "www.implisense.com",
           city: "Berlin",
           zip: "10115",
         }
       }
     ];
     ```
   - Connect from Config node.

4. **Create SplitInBatches Node**  
   - Name: `Split Batches`  
   - Type: SplitInBatches  
   - Parameters: Batch Size = 5  
   - Connect from Mock Lead Input.

5. **Create Wait Node for Rate Limiting**  
   - Name: `Rate Limit`  
   - Type: Wait  
   - Parameters: Wait Amount = 2 seconds  
   - Connect from Split Batches.

6. **Create HTTP Request Node for Company Lookup**  
   - Name: `Lookup Company`  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://german-company-data.p.rapidapi.com/lookup`  
   - Headers:  
     - `Accept`: `application/json`  
     - `x-rapidapi-host`: `german-company-data.p.rapidapi.com`  
     - `x-rapidapi-key`: `={{$('Config').item.json['x-rapidapi-key']}}`  
   - Body Parameters (as JSON):  
     - `active`: `true`  
     - `name`: `={{ $json.companyName }}`  
     - `url`: `={{ $json.domain }}`  
   - Query Parameters:  
     - `size`: `10`  
   - Connect from Rate Limit.

7. **Create HTTP Request Node for Event Retrieval**  
   - Name: `Get Events`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://german-company-data.p.rapidapi.com/companies/{{ $json.companies[0].id }}/events`  
   - Headers:  
     - `x-rapidapi-host`: `german-company-data.p.rapidapi.com`  
     - `x-rapidapi-key`: `={{$('Config').item.json['x-rapidapi-key']}}`  
   - Query Parameters:  
     - `category`: `MANAGEMENT_AND_TEAM,FINANCES_AND_CAPITAL,NEWS_AND_EVENTS`  
     - `since`: `2017-01-01`  
     - `size`: `100`  
   - Connect from Lookup Company.

8. **Create If Node to Check API Success**  
   - Name: `API Success?`  
   - Type: If  
   - Condition: Number, `$json.statusCode === 200`  
   - Connect from Get Events.

9. **Create Code Node to Normalize Events**  
   - Name: `Normalize Events`  
   - Type: Code  
   - Code:  
     ```javascript
     const response = $input.item.json;
     const companyId = response.company_id;
     const companyName = response.company_name;
     const workflowRunId = response.workflow_run_id;
     const eventsData = response.body || {};
     const events = eventsData.events || [];
     if (events.length === 0) return [];
     return events.map((event, idx) => ({
       json: {
         event_idx: idx,
         company_id: companyId,
         company_name: companyName,
         event_type: event.type || 'unknown',
         event_category: (event.category && event.category.code) || event.category || null,
         event_title: event.title || '',
         event_text: event.text || '',
         event_timestamp: event.timestamp || new Date().toISOString(),
         event_source: event.source || null,
         workflow_run_id: workflowRunId,
         fetched_at: new Date().toISOString()
       }
     }));
     ```
   - Connect from API Success? (true branch).

10. **Create Code Node to Log API Errors**  
    - Name: `Log API Error`  
    - Type: Code  
    - Code:  
      ```javascript
      const error = $input.item.json.error || 'Unknown error';
      const statusCode = $input.item.json.statusCode || 0;
      console.error('API failed: ' + error);
      return {
        json: {
          company_id: $input.item.json.company_id,
          company_name: $input.item.json.company_name,
          error_type: 'api_failed',
          error_message: error,
          status_code: statusCode
        }
      };
      ```
    - Connect from API Success? (false branch).

11. **Create Merge Node to Combine Normalized Events & Errors**  
    - Name: `Merge Results`  
    - Type: Merge  
    - Mode: Combine  
    - Include Unpaired: true  
    - Combination Mode: Merge By Position  
    - Connect from Normalize Events and Log API Error.

12. **Create If Node to Filter Relevant Events**  
    - Name: `Filter Relevant`  
    - Type: If  
    - Conditions:  
      - String: `$json.event_category` is Not Empty  
      - String: `$json.event_category` matches regex `MANAGEMENT_AND_TEAM|FINANCES_AND_CAPITAL|MERGERS_AND_ACQUISITIONS|BANKRUPTCY|INSOLVENCY|EXPANSION|RELOCATION`  
    - Connect from Merge Results.

13. **Create RemoveDuplicates Node**  
    - Name: `Deduplicate`  
    - Type: RemoveDuplicates  
    - Connect from Filter Relevant (true branch).

14. **Create Code Node to Prepare Notification**  
    - Name: `Prepare Notification`  
    - Type: Code  
    - Code:  
      ```javascript
      const event = $input.item.json;
      function getUrgency(category) {
        if (['BANKRUPTCY', 'INSOLVENCY', 'MERGERS_AND_ACQUISITIONS'].includes(category)) return 'HIGH';
        if (['MANAGEMENT_AND_TEAM', 'FINANCES_AND_CAPITAL'].includes(category)) return 'MEDIUM';
        return 'LOW';
      }
      const eventDate = new Date(event.event_timestamp);
      const ageHours = Math.floor((Date.now() - eventDate.getTime()) / (1000 * 60 * 60));
      const urgency = getUrgency(event.event_category);
      const message = urgency + ' Priority Event\n\n' +
        'Company: ' + event.company_name + ' (' + event.company_id + ')\n' +
        'Category: ' + event.event_category + '\n' +
        'Title: ' + event.event_title + '\n' +
        'Time: ' + event.event_timestamp + ' (' + ageHours + 'h ago)\n' +
        'Source: ' + (event.event_source || 'N/A') + '\n\n' +
        (event.event_text || 'No description');
      return {
        json: {
          company_name: event.company_name,
          company_id: event.company_id,
          event_title: event.event_title,
          event_category: event.event_category,
          event_timestamp: event.event_timestamp,
          urgency: urgency,
          event_age_hours: ageHours,
          formatted_message: message,
          workflow_run_id: event.workflow_run_id
        }
      };
      ```
    - Connect from Deduplicate.

15. **Create Placeholder NoOp Node for Notification Sending**  
    - Name: `Email, Chat, Webhook etc.`  
    - Type: NoOp  
    - Connect from Prepare Notification.

16. **Create If Node for Notification Sent Check (disabled)**  
    - Name: `Notification Sent?`  
    - Type: If  
    - Condition: Check if `$json.error` is empty (string is empty)  
    - Disabled: true (optional)  
    - Connect from Email, Chat, Webhook etc.

17. **Create Code Node for Logging Success**  
    - Name: `Log Success`  
    - Type: Code  
    - Code:  
      ```javascript
      console.log('Sent: ' + $input.item.json.company_name + ' - ' + $input.item.json.event_title);
      return { json: { ...$input.item.json, status: 'sent' } };
      ```
    - Connect from Notification Sent? (true branch).

18. **Create Code Node for Logging Failure**  
    - Name: `Log Failed`  
    - Type: Code  
    - Code:  
      ```javascript
      console.error('Failed: ' + ($input.item.json.error || 'Unknown'));
      return { json: { ...$input.item.json, status: 'failed' } };
      ```
    - Connect from Notification Sent? (false branch).

19. **Create Merge Node to Combine Logs**  
    - Name: `Merge Notifications`  
    - Type: Merge  
    - Mode: Combine  
    - Connect from Log Success and Log Failed.

20. **Create NoOp Node for Loop Control**  
    - Name: `Loop Continue`  
    - Type: NoOp  
    - Connect from Merge Notifications.  
    - Connect back to Split Batches (to continue next batch).

21. **Create Code Node for Summary**  
    - Name: `Create Summary`  
    - Type: Code  
    - Code:  
      ```javascript
      const items = $input.all();
      const total = items.length;
      const sent = items.filter(i => i.json.status === 'sent').length;
      const failed = total - sent;
      const summary = {
        workflow_run_id: $('Config').first().json.workflow_run_id || 'unknown',
        total_events: total,
        notifications_sent: sent,
        notifications_failed: failed,
        success_rate: total > 0 ? Math.round((sent / total) * 100) : 0,
        executed_at: new Date().toISOString()
      };
      console.log('Summary: ' + sent + '/' + total + ' sent');
      return { json: summary };
      ```
    - Connect from Split Batches (final output).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                          | Context or Link                                                                                          |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------|
| # B2B Register Event Monitoring for German Companies: This n8n workflow monitors significant register, financial, and news-related events for German companies using Implisense API. Setup steps include replacing mock data with actual lead sources, configuring RapidAPI credentials, and adjusting notification outputs. | See sticky note "🏗️ Architecture Notes" in workflow. RapidAPI Implisense API: https://rapidapi.com/Implisense/api/German%20Company%20Data |
| Replace the `Mock Lead Input` node with your actual lead source such as a CRM connector (Salesforce, HubSpot), a database query, or a CSV import.                                                                                                                                          | User customization instruction.                                                                         |
| Insert your RapidAPI `x-rapidapi-key` into the `Config` node for authentication.                                                                                                                                                                      | RapidAPI Account required, free tier available.                                                         |
| The notification sending block is a placeholder using a NoOp node; replace it with your preferred notification channel nodes such as Email, Slack, or Webhook.                                                                                      | User customization instruction.                                                                         |

---

**Disclaimer:**  
The provided text derives exclusively from an n8n workflow automation. All data processed is legal and publicly available, and the workflow complies with content policies.