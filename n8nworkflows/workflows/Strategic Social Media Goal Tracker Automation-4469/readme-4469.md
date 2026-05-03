Strategic Social Media Goal Tracker Automation

https://n8nworkflows.xyz/workflows/strategic-social-media-goal-tracker-automation-4469


# Strategic Social Media Goal Tracker Automation

### 1. Workflow Overview

The **"Social Media Strategy Alignment Workflow"** automates the management and tracking of strategic social media goals for clients. It is designed to intake client data, create measurable SMART goals, collect and process social media performance data across multiple platforms, monitor alert thresholds and milestones, generate periodic reports, and provide content calendars on demand.

The workflow is structured into the following logical blocks:

- **1.1 Client Intake and Goal Setup:** Handles initial client data submission, SMART goal creation, and client profile storage.
- **1.2 Scheduled Data Collection and Processing:** Periodically retrieves social media performance data for all clients from Instagram, Facebook, LinkedIn, and Twitter/X, processes it, and stores it.
- **1.3 Alerting and Milestone Notifications:** After data processing, checks if alert thresholds or milestones are met and sends notification emails accordingly.
- **1.4 Monthly Reporting:** Monthly scheduled generation of performance reports per client, formatted in HTML and sent via email.
- **1.5 Quarterly Strategy Review:** Quarterly scheduled generation of strategic review documents for clients, prepared and emailed.
- **1.6 On-Demand Content Calendar Generation:** Webhook-triggered generation of client-specific content calendars returned as HTML.

---

### 2. Block-by-Block Analysis

#### 2.1 Client Intake and Goal Setup

- **Overview:** Accepts new client data via webhook, generates SMART goals from the input, and stores the client profile in the database.
- **Nodes Involved:**  
  - Client Intake Form (Webhook)  
  - Create SMART Goals (Function)  
  - Store Client Profile (MongoDB)

##### Nodes:

- **Client Intake Form**  
  - Type: Webhook  
  - Role: Entry point for new client data submission.  
  - Config: Exposes a webhook endpoint identified by `client-intake-form`.  
  - Inputs: External HTTP POST requests.  
  - Outputs: Passes data to "Create SMART Goals".  
  - Failure Modes: Network downtime, invalid payloads, webhook misconfiguration.

- **Create SMART Goals**  
  - Type: Function  
  - Role: Processes client input to create SMART (Specific, Measurable, Achievable, Relevant, Time-bound) goals.  
  - Config: Custom JavaScript code transforming form data into structured goals.  
  - Inputs: Data from "Client Intake Form".  
  - Outputs: Structured goals to "Store Client Profile".  
  - Failure Modes: Code errors, missing or malformed input fields.

- **Store Client Profile**  
  - Type: MongoDB  
  - Role: Persists client profile and goals into the database.  
  - Config: MongoDB credentials must be configured; parameters include collection and database names.  
  - Inputs: SMART goals and client data.  
  - Outputs: None (terminal in this block).  
  - Failure Modes: Database connectivity issues, write failures, credential errors.

---

#### 2.2 Scheduled Data Collection and Processing

- **Overview:** Automatically triggered daily to collect social media data for all clients, split by client, then conditionally request data from Instagram, Facebook, LinkedIn, and Twitter/X. The data is processed and stored for further use.
- **Nodes Involved:**  
  - Daily Data Collection (Cron)  
  - Get All Clients (MongoDB)  
  - Split By Client (SplitInBatches)  
  - Has Instagram? (If)  
  - Instagram Data (Instagram)  
  - Has Facebook? (If)  
  - Facebook Data (Facebook Graph API)  
  - Has LinkedIn? (If)  
  - LinkedIn Data (LinkedIn Companies)  
  - Has Twitter/X? (If)  
  - Twitter/X Data (Twitter)  
  - Process Performance Data (Function)  
  - Store Performance Data (MongoDB)

##### Nodes:

- **Daily Data Collection**  
  - Type: Cron  
  - Role: Triggers daily data collection at a scheduled time.  
  - Config: Cron expression for daily execution.  
  - Inputs: None.  
  - Outputs: Triggers "Get All Clients".  
  - Failure Modes: Cron misconfigurations, scheduler downtime.

- **Get All Clients**  
  - Type: MongoDB  
  - Role: Retrieves all client profiles from database for data collection.  
  - Config: Database and collection settings; reads all client documents.  
  - Inputs: Trigger from "Daily Data Collection".  
  - Outputs: Data to "Split By Client".  
  - Failure Modes: Database read errors.

- **Split By Client**  
  - Type: SplitInBatches  
  - Role: Splits client list into batches for processing one client at a time.  
  - Config: Batch size configured (usually 1).  
  - Inputs: Client list from "Get All Clients".  
  - Outputs: Four parallel conditional paths for social media platform checks.  
  - Failure Modes: Batch misconfiguration, large data sets causing delays.

- **Has Instagram? / Has Facebook? / Has LinkedIn? / Has Twitter/X?**  
  - Type: If  
  - Role: Checks if client has a profile on the respective platform.  
  - Config: Conditional expressions checking client profile fields for platform presence.  
  - Inputs: Single client data from "Split By Client".  
  - Outputs: If true, fetch platform data; if false, skip to "Process Performance Data".  
  - Failure Modes: Missing fields, incorrect data formats.

- **Instagram Data / Facebook Data / LinkedIn Data / Twitter/X Data**  
  - Type: Platform-specific API nodes  
  - Role: Retrieves social media performance data for the client.  
  - Config: API credentials for each platform required; queries configured for relevant metrics.  
  - Inputs: Client profile with access tokens or identifiers.  
  - Outputs: Raw performance data to "Process Performance Data".  
  - Failure Modes: Auth token expiry, API rate limits, network issues, invalid client credentials.

- **Process Performance Data**  
  - Type: Function  
  - Role: Aggregates and formats collected data into a unified structure for storage.  
  - Config: JavaScript function for data normalization and calculations.  
  - Inputs: Social media data from all platform nodes and fallback paths.  
  - Outputs: Processed records to "Store Performance Data".  
  - Failure Modes: Data inconsistencies, function errors, missing data fields.

- **Store Performance Data**  
  - Type: MongoDB  
  - Role: Saves processed performance data into the database for historic tracking.  
  - Config: Database connection parameters and collection names.  
  - Inputs: Processed data from "Process Performance Data".  
  - Outputs: To alert and milestone checks.  
  - Failure Modes: Write failures, database unavailability.

---

#### 2.3 Alerting and Milestone Notifications

- **Overview:** Evaluates stored performance data against predefined alert thresholds and milestones. Sends notification emails if any conditions are met.
- **Nodes Involved:**  
  - Check Alert Thresholds (If)  
  - Send Alert Email (EmailSend)  
  - Check Milestone Achievements (If)  
  - Send Milestone Email (EmailSend)

##### Nodes:

- **Check Alert Thresholds**  
  - Type: If  
  - Role: Determines if performance metrics surpass alert thresholds.  
  - Config: Conditional expressions referencing threshold values.  
  - Inputs: Data from "Store Performance Data".  
  - Outputs: Sends alerts if true; terminates if false.  
  - Failure Modes: Logic errors, missing threshold data.

- **Send Alert Email**  
  - Type: EmailSend  
  - Role: Sends alert notification emails to stakeholders.  
  - Config: SMTP or email service credentials configured; email template and recipients specified.  
  - Inputs: Alert condition triggered data.  
  - Outputs: None.  
  - Failure Modes: Email server errors, invalid recipient addresses.

- **Check Milestone Achievements**  
  - Type: If  
  - Role: Checks if clients have reached key milestones in their social media goals.  
  - Config: Conditional logic on milestone criteria.  
  - Inputs: Performance data from "Store Performance Data".  
  - Outputs: Sends milestone emails if true.  
  - Failure Modes: Same as alert checks.

- **Send Milestone Email**  
  - Type: EmailSend  
  - Role: Sends congratulatory or milestone achievement emails.  
  - Config: Email service credentials, templates, and recipients.  
  - Inputs: Milestone condition true data.  
  - Outputs: None.  
  - Failure Modes: Same as "Send Alert Email".

---

#### 2.4 Monthly Reporting

- **Overview:** Scheduled monthly process to generate detailed performance reports for clients and send via email.
- **Nodes Involved:**  
  - Monthly Report Trigger (Cron)  
  - Get Clients for Reports (MongoDB)  
  - Split By Client for Reports (SplitInBatches)  
  - Get Client Performance History (MongoDB)  
  - Generate Monthly Report (Function)  
  - Create HTML Report (Function)  
  - Send Monthly Report (EmailSend)

##### Nodes:

- **Monthly Report Trigger**  
  - Type: Cron  
  - Role: Schedules the monthly report generation.  
  - Config: Cron expression for monthly execution.  
  - Inputs: None.  
  - Outputs: Triggers "Get Clients for Reports".  
  - Failure Modes: Cron misconfiguration.

- **Get Clients for Reports**  
  - Type: MongoDB  
  - Role: Retrieves all clients for whom reports are generated.  
  - Config: Database read parameters.  
  - Inputs: Trigger from cron.  
  - Outputs: Client data to "Split By Client for Reports".  
  - Failure Modes: DB read errors.

- **Split By Client for Reports**  
  - Type: SplitInBatches  
  - Role: Processes clients one at a time for report generation.  
  - Config: Batch size set to 1.  
  - Inputs: Client list.  
  - Outputs: To "Get Client Performance History".  
  - Failure Modes: Large batch sizes causing delays.

- **Get Client Performance History**  
  - Type: MongoDB  
  - Role: Retrieves historic performance data for each client.  
  - Config: Query parameters for historical data.  
  - Inputs: Single client from batch.  
  - Outputs: Data to "Generate Monthly Report".  
  - Failure Modes: Query failures.

- **Generate Monthly Report**  
  - Type: Function  
  - Role: Aggregates historical data into a report format.  
  - Config: JavaScript logic for summarization.  
  - Inputs: Performance history data.  
  - Outputs: Report data to "Create HTML Report".  
  - Failure Modes: Code errors, missing data.

- **Create HTML Report**  
  - Type: Function  
  - Role: Converts report data into HTML format for email.  
  - Config: HTML template logic in JavaScript.  
  - Inputs: Report data.  
  - Outputs: HTML content to "Send Monthly Report".  
  - Failure Modes: Template errors.

- **Send Monthly Report**  
  - Type: EmailSend  
  - Role: Emails the HTML report to clients or internal stakeholders.  
  - Config: Email credentials and template settings.  
  - Inputs: HTML report content.  
  - Outputs: None.  
  - Failure Modes: Email delivery failures.

---

#### 2.5 Quarterly Strategy Review

- **Overview:** Triggers quarterly to compile and send strategic review documents based on accumulated data.
- **Nodes Involved:**  
  - Quarterly Strategy Review (Cron)  
  - Get Clients for Strategy Review (MongoDB)  
  - Split By Client for Review (SplitInBatches)  
  - Get Quarterly Performance Data (MongoDB)  
  - Generate Strategy Review (Function)  
  - Create Strategy Review Document (Function)  
  - Send Strategy Review (EmailSend)

##### Nodes:

- **Quarterly Strategy Review**  
  - Type: Cron  
  - Role: Triggers the quarterly review process.  
  - Config: Cron expression for quarterly execution.  
  - Inputs: None.  
  - Outputs: To "Get Clients for Strategy Review".  
  - Failure Modes: Scheduler errors.

- **Get Clients for Strategy Review**  
  - Type: MongoDB  
  - Role: Fetches clients for whom quarterly reviews are prepared.  
  - Config: Query parameters.  
  - Inputs: Trigger from cron.  
  - Outputs: To "Split By Client for Review".  
  - Failure Modes: DB read errors.

- **Split By Client for Review**  
  - Type: SplitInBatches  
  - Role: Processes clients individually for review generation.  
  - Config: Batch size set to 1.  
  - Inputs: Client list.  
  - Outputs: To "Get Quarterly Performance Data".  
  - Failure Modes: Large batch delays.

- **Get Quarterly Performance Data**  
  - Type: MongoDB  
  - Role: Retrieves quarterly performance data per client.  
  - Config: Query parameters.  
  - Inputs: Client data.  
  - Outputs: To "Generate Strategy Review".  
  - Failure Modes: Query or DB failures.

- **Generate Strategy Review**  
  - Type: Function  
  - Role: Creates structured review content from data.  
  - Config: JavaScript logic.  
  - Inputs: Quarterly data.  
  - Outputs: To "Create Strategy Review Document".  
  - Failure Modes: Code errors.

- **Create Strategy Review Document**  
  - Type: Function  
  - Role: Formats review into document format (likely HTML or PDF).  
  - Config: Template logic.  
  - Inputs: Review content.  
  - Outputs: To "Send Strategy Review".  
  - Failure Modes: Formatting issues.

- **Send Strategy Review**  
  - Type: EmailSend  
  - Role: Sends the final strategy review document via email.  
  - Config: Email credentials, templates.  
  - Inputs: Document content.  
  - Outputs: None.  
  - Failure Modes: Email sending issues.

---

#### 2.6 On-Demand Content Calendar Generation

- **Overview:** Accepts webhook requests to generate and return a client-specific content calendar in HTML format.
- **Nodes Involved:**  
  - Content Calendar Webhook (Webhook)  
  - Get Client for Content Calendar (MongoDB)  
  - Generate Content Calendar (Function)  
  - Create Content Calendar HTML (Function)  
  - Return Content Calendar (RespondToWebhook)

##### Nodes:

- **Content Calendar Webhook**  
  - Type: Webhook  
  - Role: Entry point for content calendar request.  
  - Config: Webhook ID `content-calendar-webhook`.  
  - Inputs: HTTP requests with client identifier.  
  - Outputs: To "Get Client for Content Calendar".  
  - Failure Modes: Network issues, invalid request data.

- **Get Client for Content Calendar**  
  - Type: MongoDB  
  - Role: Retrieves client profile needed for calendar generation.  
  - Config: Query by client ID from webhook input.  
  - Inputs: Webhook data.  
  - Outputs: To "Generate Content Calendar".  
  - Failure Modes: Client not found, DB errors.

- **Generate Content Calendar**  
  - Type: Function  
  - Role: Creates a content calendar structure based on client profile and strategy.  
  - Config: JavaScript logic generating calendar entries.  
  - Inputs: Client data.  
  - Outputs: To "Create Content Calendar HTML".  
  - Failure Modes: Logic errors.

- **Create Content Calendar HTML**  
  - Type: Function  
  - Role: Converts calendar structure into HTML.  
  - Config: Template logic for calendar formatting.  
  - Inputs: Calendar data.  
  - Outputs: To "Return Content Calendar".  
  - Failure Modes: HTML formatting issues.

- **Return Content Calendar**  
  - Type: RespondToWebhook  
  - Role: Sends the generated content calendar HTML back as HTTP response.  
  - Config: Output settings to respond with HTML content.  
  - Inputs: HTML content.  
  - Outputs: HTTP response.  
  - Failure Modes: Response timeouts, invalid response formatting.

---

### 3. Summary Table

| Node Name                      | Node Type              | Functional Role                                  | Input Node(s)                                | Output Node(s)                        | Sticky Note                 |
|--------------------------------|------------------------|------------------------------------------------|----------------------------------------------|-------------------------------------|-----------------------------|
| Client Intake Form              | Webhook                | Receives new client intake data                 | -                                            | Create SMART Goals                   |                             |
| Create SMART Goals             | Function               | Generates SMART goals from intake data          | Client Intake Form                            | Store Client Profile                 |                             |
| Store Client Profile           | MongoDB                | Stores client profile and goals                  | Create SMART Goals                            | -                                   |                             |
| Daily Data Collection          | Cron                   | Triggers daily social media data collection      | -                                            | Get All Clients                     |                             |
| Get All Clients               | MongoDB                | Retrieves all clients                             | Daily Data Collection                         | Split By Client                     |                             |
| Split By Client               | SplitInBatches         | Splits client list for individual processing     | Get All Clients                               | Has Instagram?, Has Facebook?, Has LinkedIn?, Has Twitter/X? |                             |
| Has Instagram?                | If                     | Checks if client has Instagram                    | Split By Client                               | Instagram Data / Process Performance Data |                             |
| Instagram Data                | Instagram              | Fetches Instagram data                            | Has Instagram?                                | Process Performance Data             |                             |
| Has Facebook?                 | If                     | Checks if client has Facebook                      | Split By Client                               | Facebook Data / Process Performance Data |                             |
| Facebook Data                | Facebook Graph API      | Fetches Facebook data                             | Has Facebook?                                 | Process Performance Data             |                             |
| Has LinkedIn?                 | If                     | Checks if client has LinkedIn                      | Split By Client                               | LinkedIn Data / Process Performance Data |                             |
| LinkedIn Data                | LinkedIn Companies      | Fetches LinkedIn company data                     | Has LinkedIn?                                 | Process Performance Data             |                             |
| Has Twitter/X?                | If                     | Checks if client has Twitter/X                     | Split By Client                               | Twitter/X Data / Process Performance Data |                             |
| Twitter/X Data               | Twitter                 | Fetches Twitter/X data                            | Has Twitter/X?                                | Process Performance Data             |                             |
| Process Performance Data       | Function               | Aggregates and normalizes social media data      | Instagram Data, Facebook Data, LinkedIn Data, Twitter/X Data, Has X? false paths | Store Performance Data               |                             |
| Store Performance Data         | MongoDB                | Stores processed performance data                 | Process Performance Data                       | Check Alert Thresholds, Check Milestone Achievements |                             |
| Check Alert Thresholds         | If                     | Checks if alerts conditions are met               | Store Performance Data                         | Send Alert Email / None              |                             |
| Send Alert Email              | EmailSend              | Sends alert notifications                         | Check Alert Thresholds                         | -                                   |                             |
| Check Milestone Achievements   | If                     | Checks if milestones are achieved                  | Store Performance Data                         | Send Milestone Email / None          |                             |
| Send Milestone Email          | EmailSend              | Sends milestone achievement emails                | Check Milestone Achievements                   | -                                   |                             |
| Monthly Report Trigger         | Cron                   | Triggers monthly report generation                 | -                                            | Get Clients for Reports             |                             |
| Get Clients for Reports        | MongoDB                | Retrieves clients for monthly reports              | Monthly Report Trigger                         | Split By Client for Reports          |                             |
| Split By Client for Reports    | SplitInBatches         | Processes clients individually for reports         | Get Clients for Reports                        | Get Client Performance History      |                             |
| Get Client Performance History | MongoDB                | Fetches historic performance data                   | Split By Client for Reports                    | Generate Monthly Report             |                             |
| Generate Monthly Report        | Function               | Summarizes performance history into report data    | Get Client Performance History                 | Create HTML Report                  |                             |
| Create HTML Report             | Function               | Formats report data as HTML                          | Generate Monthly Report                        | Send Monthly Report                 |                             |
| Send Monthly Report           | EmailSend              | Emails monthly HTML report                          | Create HTML Report                             | -                                   |                             |
| Quarterly Strategy Review      | Cron                   | Triggers quarterly strategy review                  | -                                            | Get Clients for Strategy Review     |                             |
| Get Clients for Strategy Review| MongoDB                | Retrieves clients for quarterly review              | Quarterly Strategy Review                      | Split By Client for Review           |                             |
| Split By Client for Review     | SplitInBatches         | Processes clients individually for review           | Get Clients for Strategy Review                | Get Quarterly Performance Data      |                             |
| Get Quarterly Performance Data | MongoDB                | Fetches quarterly data                               | Split By Client for Review                      | Generate Strategy Review            |                             |
| Generate Strategy Review       | Function               | Creates structured review content                     | Get Quarterly Performance Data                 | Create Strategy Review Document     |                             |
| Create Strategy Review Document| Function               | Formats review content into document                   | Generate Strategy Review                        | Send Strategy Review                |                             |
| Send Strategy Review          | EmailSend              | Sends strategy review document                        | Create Strategy Review Document                 | -                                   |                             |
| Content Calendar Webhook       | Webhook                | Receives content calendar generation requests         | -                                            | Get Client for Content Calendar     |                             |
| Get Client for Content Calendar| MongoDB                | Retrieves client profile for calendar generation       | Content Calendar Webhook                        | Generate Content Calendar           |                             |
| Generate Content Calendar      | Function               | Generates calendar data structure                       | Get Client for Content Calendar                 | Create Content Calendar HTML        |                             |
| Create Content Calendar HTML   | Function               | Converts calendar data to HTML                           | Generate Content Calendar                       | Return Content Calendar             |                             |
| Return Content Calendar        | RespondToWebhook       | Returns generated calendar HTML via webhook response    | Create Content Calendar HTML                    | -                                   |                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node: "Client Intake Form"**  
   - Type: Webhook  
   - Configure path: `/client-intake-form`  
   - Method: POST  
   - No authentication required unless desired.

2. **Create Function Node: "Create SMART Goals"**  
   - Connect from "Client Intake Form"  
   - Add JavaScript code to parse intake data and create SMART goals object.  
   - Output structured goals.

3. **Create MongoDB Node: "Store Client Profile"**  
   - Connect from "Create SMART Goals"  
   - Configure MongoDB credentials and specify database and collection for client profiles.  
   - Operation: Insert or Upsert client data with SMART goals.

4. **Create Cron Node: "Daily Data Collection"**  
   - Configure to run daily at desired time (e.g., 00:00).  
   - Connect to "Get All Clients".

5. **Create MongoDB Node: "Get All Clients"**  
   - Connect from "Daily Data Collection"  
   - Configure to fetch all client documents.

6. **Create SplitInBatches Node: "Split By Client"**  
   - Connect from "Get All Clients"  
   - Batch size: 1 (process one client at a time).

7. **Create If Nodes for Social Platforms:**  
   - "Has Instagram?"  
     - Condition: check if Instagram field exists in client data.  
   - "Has Facebook?"  
     - Condition: check if Facebook field exists.  
   - "Has LinkedIn?"  
     - Condition: check if LinkedIn field exists.  
   - "Has Twitter/X?"  
     - Condition: check if Twitter/X field exists.

8. **Create Social Media API Nodes:**  
   - "Instagram Data" (Instagram node)  
     - Connect from "Has Instagram?" true output  
     - Configure with Instagram API credentials and query metrics.  
   - "Facebook Data" (Facebook Graph API)  
     - Connect from "Has Facebook?" true output  
     - Configure Facebook API credentials and queries.  
   - "LinkedIn Data" (LinkedIn Companies)  
     - Connect from "Has LinkedIn?" true output  
     - Configure LinkedIn API credentials.  
   - "Twitter/X Data" (Twitter node)  
     - Connect from "Has Twitter/X?" true output  
     - Configure Twitter API credentials.

9. **Create Function Node: "Process Performance Data"**  
   - Connect all social media nodes true outputs to this node, and all false outputs from if nodes as well (for clients without profiles).  
   - Implement JavaScript logic to normalize and aggregate data from all platforms.

10. **Create MongoDB Node: "Store Performance Data"**  
    - Connect from "Process Performance Data"  
    - Configure for performance data collection.

11. **Create If Node: "Check Alert Thresholds"**  
    - Connect from "Store Performance Data"  
    - Condition: check if any metric exceeds alert thresholds.

12. **Create Email Send Node: "Send Alert Email"**  
    - Connect from "Check Alert Thresholds" true output  
    - Configure SMTP or email service credentials, recipients, subject, and body templates.

13. **Create If Node: "Check Milestone Achievements"**  
    - Connect from "Store Performance Data"  
    - Condition: check if milestones are reached.

14. **Create Email Send Node: "Send Milestone Email"**  
    - Connect from "Check Milestone Achievements" true output  
    - Configure similar to alert email.

15. **Create Cron Node: "Monthly Report Trigger"**  
    - Configure monthly schedule.  
    - Connect to "Get Clients for Reports".

16. **Create MongoDB Node: "Get Clients for Reports"**  
    - Connect from "Monthly Report Trigger"  
    - Fetch clients for reporting.

17. **Create SplitInBatches Node: "Split By Client for Reports"**  
    - Connect from "Get Clients for Reports"  
    - Batch size: 1.

18. **Create MongoDB Node: "Get Client Performance History"**  
    - Connect from "Split By Client for Reports"  
    - Query historic performance data.

19. **Create Function Node: "Generate Monthly Report"**  
    - Connect from "Get Client Performance History"  
    - Aggregate data into report format.

20. **Create Function Node: "Create HTML Report"**  
    - Connect from "Generate Monthly Report"  
    - Format report as HTML.

21. **Create Email Send Node: "Send Monthly Report"**  
    - Connect from "Create HTML Report"  
    - Configure email credentials, recipients, and content.

22. **Create Cron Node: "Quarterly Strategy Review"**  
    - Configure quarterly schedule.  
    - Connect to "Get Clients for Strategy Review".

23. **Create MongoDB Node: "Get Clients for Strategy Review"**  
    - Connect from "Quarterly Strategy Review"  
    - Fetch clients.

24. **Create SplitInBatches Node: "Split By Client for Review"**  
    - Connect from "Get Clients for Strategy Review"  
    - Batch size: 1.

25. **Create MongoDB Node: "Get Quarterly Performance Data"**  
    - Connect from "Split By Client for Review"  
    - Fetch quarterly data.

26. **Create Function Node: "Generate Strategy Review"**  
    - Connect from "Get Quarterly Performance Data"  
    - Prepare structured review content.

27. **Create Function Node: "Create Strategy Review Document"**  
    - Connect from "Generate Strategy Review"  
    - Format as document (HTML or PDF).

28. **Create Email Send Node: "Send Strategy Review"**  
    - Connect from "Create Strategy Review Document"  
    - Configure email settings.

29. **Create Webhook Node: "Content Calendar Webhook"**  
    - Configure webhook path `/content-calendar-webhook`.

30. **Create MongoDB Node: "Get Client for Content Calendar"**  
    - Connect from "Content Calendar Webhook"  
    - Fetch client profile by ID.

31. **Create Function Node: "Generate Content Calendar"**  
    - Connect from "Get Client for Content Calendar"  
    - Generate calendar data.

32. **Create Function Node: "Create Content Calendar HTML"**  
    - Connect from "Generate Content Calendar"  
    - Format as HTML.

33. **Create RespondToWebhook Node: "Return Content Calendar"**  
    - Connect from "Create Content Calendar HTML"  
    - Return HTML content as webhook response.

34. **Configure credentials:**  
    - MongoDB with appropriate read/write permissions.  
    - Instagram, Facebook, LinkedIn, Twitter API credentials with necessary scopes.  
    - Email service (SMTP, SendGrid, etc.) with verified sender and recipient settings.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow automates multi-platform social media strategy tracking with goal alignment.     | Workflow description                                                                                |
| Use valid API credentials and maintain token refresh mechanisms to avoid auth failures.        | API nodes (Instagram, Facebook, LinkedIn, Twitter)                                                 |
| Ensure MongoDB collections have necessary indexes for efficient queries on client and date fields. | MongoDB node optimization                                                                           |
| Email nodes require SMTP or third-party email service credentials with proper permission setups.| EmailSend nodes                                                                                     |
| Webhook nodes allow external integrations to trigger workflows or provide data inputs.         | Client Intake Form, Content Calendar Webhook                                                        |
| Cron nodes require n8n instance scheduler operational and time zone considerations.             | Scheduled triggers                                                                                  |
| Consider implementing error handling (e.g., try/catch in function nodes, node error triggers)  | To increase workflow robustness                                                                    |
| For complex data processing, unit test JavaScript functions separately before deployment.      | Function nodes (Create SMART Goals, Process Performance Data, Report Generators)                    |

---

**Disclaimer:** The provided content is exclusively derived from an automated n8n workflow. It complies with all applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.