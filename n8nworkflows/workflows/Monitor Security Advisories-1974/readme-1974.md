Monitor Security Advisories

https://n8nworkflows.xyz/workflows/monitor-security-advisories-1974


# Monitor Security Advisories

### 1. Workflow Overview

This workflow automates the monitoring and notification of Palo Alto Networks security advisories, specifically targeting the GlobalProtect and Traps product lines. It fetches the latest advisories from Palo Alto’s RSS feed, filters advisories posted within the last 24 hours, categorizes them by product, and routes them through distinct processing paths. The GlobalProtect advisories result in Jira issue creation for incident tracking, while Traps advisories trigger email notifications to customers or stakeholders.

The workflow is designed to be triggered manually or scheduled to run daily at 1 AM. It includes filtering mechanisms to avoid stale advisories, dynamic extraction of advisory metadata, and integration with Jira and Gmail for issue tracking and communications. The design accommodates customization for other Palo Alto products or different incident management systems.

**Logical Blocks:**

- **1.1 Input Reception:** Manual or scheduled triggers start the workflow.
- **1.2 Data Acquisition:** Retrieves Palo Alto Networks security advisories via RSS feed.
- **1.3 Data Extraction & Filtering:** Extracts advisory metadata and filters advisories posted in the last 24 hours.
- **1.4 Advisory Categorization:** Splits advisories into GlobalProtect and Traps paths based on advisory type.
- **1.5 Incident Management (GlobalProtect path):** Creates Jira issues for GlobalProtect advisories.
- **1.6 Notification (Traps path):** Retrieves customer emails and sends email notifications.
- **1.7 Supporting Utilities:** Includes nodes for sample customer data and notes on deduplication and scheduling.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block triggers the workflow either manually via UI or automatically on a schedule.
- **Nodes Involved:**  
  - When clicking "Execute Workflow" (Manual Trigger)  
  - Run workflow every 24 hours at 1am (Schedule Trigger)

- **Node Details:**

  1. **When clicking "Execute Workflow"**  
     - Type: Manual Trigger  
     - Role: Allows manual initiation of the workflow from the n8n UI.  
     - Configuration: Default, no parameters set.  
     - Inputs: None  
     - Outputs: Connects to "Get Palo Alto security advisories" node.  
     - Edge Cases: None significant; manual trigger requires user action.

  2. **Run workflow every 24 hours at 1am**  
     - Type: Schedule Trigger  
     - Role: Automatically triggers the workflow daily at 1 AM server time.  
     - Configuration: Set to trigger at hour 1 daily.  
     - Inputs: None  
     - Outputs: Connects to "Get Palo Alto security advisories" node.  
     - Edge Cases: Time zone configuration may affect trigger time; ensure server time matches desired zone.

#### 1.2 Data Acquisition

- **Overview:** Fetches the latest Palo Alto Networks security advisories via their official RSS feed.
- **Nodes Involved:**  
  - Get Palo Alto security advisories  
  - Sticky Note1 (documentation)

- **Node Details:**

  1. **Get Palo Alto security advisories**  
     - Type: RSS Feed Read  
     - Role: Retrieves security advisories from Palo Alto’s RSS feed URL: https://security.paloaltonetworks.com/rss.xml  
     - Configuration: Default options; URL is configurable to change the source feed.  
     - Inputs: Trigger node(s)  
     - Outputs: Feeds advisory items to "Extract info" node.  
     - Edge Cases: Network connectivity issues, feed format changes, or feed downtime may cause failures.

  2. **Sticky Note1**  
     - Role: Provides explanation on the RSS node and flexibility of feed URL and extraction node.  
     - No direct technical impact.

#### 1.3 Data Extraction & Filtering

- **Overview:** Extracts structured metadata (type, subject, severity) from the advisory title and filters advisories to include only those published within the last 24 hours.
- **Nodes Involved:**  
  - Extract info  
  - Check if posted in last 24 hours (If node)  
  - Sticky Note5 (deduplication and scheduling)  
  - Ignore, stale advisory (NoOp node)

- **Node Details:**

  1. **Extract info**  
     - Type: Set  
     - Role: Parses the advisory title to extract the advisory type (e.g., GlobalProtect), subject, and severity level.  
     - Configuration: Uses JavaScript expressions with regex and string manipulation:  
       - `type`: Extracted using regex capturing the product name before colon.  
       - `subject`: Extracted subtitle before severity info.  
       - `severity`: Extracted from title, normalized to title case.  
     - Inputs: RSS feed items from "Get Palo Alto security advisories"  
     - Outputs: Passes enriched JSON to "Check if posted in last 24 hours" node.  
     - Edge Cases: Title format changes may break regex, leading to incorrect or missing fields.

  2. **Check if posted in last 24 hours**  
     - Type: If  
     - Role: Filters advisories to only those published within the last 24 hours.  
     - Configuration: Compares advisory `pubDate` field to current date minus one day using n8n date functions.  
     - Inputs: Extracted advisory info  
     - Outputs:  
       - True: Passes to product-specific filter nodes.  
       - False: Passes to "Ignore, stale advisory" node (NoOp).  
     - Edge Cases: Date parsing errors; incorrect time zones may affect filtering accuracy.

  3. **Ignore, stale advisory**  
     - Type: No Operation  
     - Role: Acts as sink for outdated advisories, effectively discarding them.  
     - Inputs: False output from "Check if posted in last 24 hours"  
     - Outputs: None  
     - Edge Cases: None.

  4. **Sticky Note5**  
     - Role: Explains the deduplication strategy based on date and synchronization with schedule trigger.  
     - Provides guidance on adjusting frequency and date filtering.

#### 1.4 Advisory Categorization

- **Overview:** Routes advisories into two branches based on whether they concern GlobalProtect or Traps products.
- **Nodes Involved:**  
  - GlobalProtect advisory? (Filter)  
  - Traps advisory? (Filter)  
  - Sticky Note4 (filtering explanation)

- **Node Details:**

  1. **GlobalProtect advisory?**  
     - Type: Filter  
     - Role: Checks if advisory title contains the string "GlobalProtect" to identify relevant advisories.  
     - Configuration: String contains condition on `$json.title` with "GlobalProtect".  
     - Inputs: True output of "Check if posted in last 24 hours"  
     - Outputs: True goes to "Create Jira issue"; default no output.  
     - Edge Cases: Case sensitivity depends on implementation; may miss advisories if title format changes.

  2. **Traps advisory?**  
     - Type: Filter  
     - Role: Checks if advisory title contains the string "Traps".  
     - Configuration: Similar string contains condition as above for "Traps".  
     - Inputs: True output of "Check if posted in last 24 hours"  
     - Outputs: True goes to "Get customers" node.  
     - Edge Cases: Same as above.

  3. **Sticky Note4**  
     - Role: Documents how to add new product filters or incident management integrations by duplicating filter nodes and connecting appropriately.

#### 1.5 Incident Management (GlobalProtect path)

- **Overview:** Creates Jira issues for GlobalProtect advisories to facilitate incident tracking and resolution.
- **Nodes Involved:**  
  - Create Jira issue  
  - Sticky Note7 (Jira explanation, merged with Sticky Note4 content in this workflow)

- **Node Details:**

  1. **Create Jira issue**  
     - Type: Jira (Jira Software Cloud API)  
     - Role: Creates an issue in a Jira project with details from the advisory.  
     - Configuration:  
       - Project and Issue Type selected dynamically (values hidden in JSON but set via list mode).  
       - Summary: Extracts substring from title starting at character 14 (skips initial prefix).  
       - Description: Includes severity, link, and publication date parsed from advisory JSON.  
       - Priority: Set from Jira list (value hidden).  
       - Credentials: Uses stored Jira credentials named "Jira Ricardo".  
     - Inputs: True output from "GlobalProtect advisory?" filter  
     - Outputs: Connects to "Get customers" node (to continue process or notify).  
     - Edge Cases: Jira API authentication failure, invalid project or issue type, API rate limiting.

#### 1.6 Notification (Traps path)

- **Overview:** Sends email notifications regarding Traps advisories to customers retrieved from a sample datastore.
- **Nodes Involved:**  
  - Get customers (Sample datastore node)  
  - Email customers (Gmail node)  
  - Sticky Note2 (customer directory explanation)  
  - Sticky Note3 (email sending explanation)

- **Node Details:**

  1. **Get customers**  
     - Type: n8nTrainingCustomerDatastore (sample dataset node)  
     - Role: Retrieves a list of customer contacts with `name` and `email` fields.  
     - Configuration: Operation set to "getAllPeople" and returns all records.  
     - Inputs: True output from "Traps advisory?" filter, and from "Create Jira issue" node (indicating parallel flows may notify customers).  
     - Outputs: Passes customer list to "Email customers" node.  
     - Edge Cases: Sample data placeholder; must be replaced with real directory integration (e.g., Google Sheets, corporate directory).  
     - See Sticky Note2 for format and replacement advice.

  2. **Email customers**  
     - Type: Gmail (OAuth2)  
     - Role: Sends email notifications to each customer about the advisory.  
     - Configuration:  
       - Recipient email: dynamically set to `$json.email` from customer data.  
       - Subject: Includes advisory type extracted from "Extract info" node.  
       - Message body: Personalized with customer first name and advisory title/link from "GlobalProtect advisory?" node’s JSON.  
       - Credentials: Uses Gmail OAuth2 named "Gmail account (David)".  
     - Inputs: Customer data from "Get customers"  
     - Outputs: None (final node)  
     - Edge Cases: Gmail API authentication, sending limits, invalid email addresses, expression failures if customer fields missing.

  3. **Sticky Note2**  
     - Role: Details how to replace the sample customer datastore with actual company email directory, e.g., via Google Sheets node.  
     - Specifies expected JSON output format.

  4. **Sticky Note3**  
     - Role: Describes the email dissemination step and how to substitute Gmail with other email providers.

#### 1.7 Supporting Utilities

- **Overview:** Provides documentation and notes to explain workflow design and usage.
- **Nodes Involved:**  
  - Sticky Note (Workflow overview and execution schedule)  
  - Sticky Note5 (Deduplication and scheduling)  
  - Sticky Note4 (Filtering explanation)  
  - Sticky Note3 (Email explanation)

- **Node Details:**

  1. **Sticky Note**  
     - Role: Describes the overall workflow purpose, filtering logic, and mention of sample email database.  
     - Includes note about execution schedule and deduplication coordination.

  2. Other sticky notes as above provide contextual documentation for respective functional blocks.

---

### 3. Summary Table

| Node Name                        | Node Type                      | Functional Role                            | Input Node(s)                        | Output Node(s)                  | Sticky Note                                                                                              |
|---------------------------------|--------------------------------|-------------------------------------------|------------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------|
| When clicking "Execute Workflow" | Manual Trigger                 | Manual start of workflow                   | None                               | Get Palo Alto security advisories |                                                                                                        |
| Run workflow every 24 hours at 1am | Schedule Trigger             | Scheduled daily start at 1 AM              | None                               | Get Palo Alto security advisories | Sticky Note: Execution Schedule; coordinate with deduplication logic                                  |
| Get Palo Alto security advisories | RSS Feed Read                | Fetches Palo Alto security advisories      | Manual Trigger, Schedule Trigger   | Extract info                    | Sticky Note1: Explains RSS node usage and customization                                                |
| Extract info                    | Set                           | Parses advisory title to extract metadata  | Get Palo Alto security advisories  | Check if posted in last 24 hours |                                                                                                        |
| Check if posted in last 24 hours | If                          | Filters advisories by publication date     | Extract info                      | GlobalProtect advisory?, Traps advisory?, Ignore, stale advisory | Sticky Note5: Deduplication explanation aligned with schedule                                         |
| GlobalProtect advisory?         | Filter                        | Filters advisories for GlobalProtect       | Check if posted in last 24 hours (true) | Create Jira issue               | Sticky Note4: Filtering and product-specific advisory explanation                                     |
| Traps advisory?                 | Filter                        | Filters advisories for Traps                | Check if posted in last 24 hours (true) | Get customers                  | Sticky Note4: Filtering and product-specific advisory explanation                                     |
| Ignore, stale advisory          | No Operation                  | Drops advisories older than 24 hours       | Check if posted in last 24 hours (false) | None                         |                                                                                                        |
| Create Jira issue               | Jira                          | Creates Jira issues for GlobalProtect advisories | GlobalProtect advisory?           | Get customers                  | Sticky Note4: Incident management and Jira integration notes                                          |
| Get customers                  | n8nTrainingCustomerDatastore  | Retrieves customer email data (sample)     | Traps advisory?, Create Jira issue | Email customers               | Sticky Note2: Explanation of customer directory node and replacement instructions                      |
| Email customers                | Gmail                         | Sends advisory email notifications          | Get customers                    | None                          | Sticky Note3: Email notification explanation and customization advice                                 |
| Sticky Note                    | Sticky Note                   | Documentation of workflow overview          | None                             | None                          | Sticky Note: Workflow overview and schedule explanation                                               |
| Sticky Note1                   | Sticky Note                   | Documentation for RSS feed node              | None                             | None                          | Sticky Note1: RSS node explanation                                                                   |
| Sticky Note2                   | Sticky Note                   | Documentation for customer directory node    | None                             | None                          | Sticky Note2: Customer directory integration guidance                                                 |
| Sticky Note3                   | Sticky Note                   | Documentation for email sending                | None                             | None                          | Sticky Note3: Email notification explanation                                                         |
| Sticky Note4                   | Sticky Note                   | Documentation for filtering nodes and Jira    | None                             | None                          | Sticky Note4: Filtering and incident management guidance                                             |
| Sticky Note5                   | Sticky Note                   | Documentation for deduplication and scheduling | None                             | None                          | Sticky Note5: Deduplication and scheduling notes                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger node:**  
   - Name: "When clicking \"Execute Workflow\""  
   - Type: Manual Trigger  
   - No parameters needed.

2. **Create Schedule Trigger node:**  
   - Name: "Run workflow every 24 hours at 1am"  
   - Type: Schedule Trigger  
   - Configure trigger to fire daily at hour 1 (1 AM) using the built-in interval settings.

3. **Create RSS Feed Read node:**  
   - Name: "Get Palo Alto security advisories"  
   - Type: RSS Feed Read  
   - Set URL to `https://security.paloaltonetworks.com/rss.xml`  
   - Use default options.

4. **Connect both trigger nodes ("When clicking Execute Workflow" and "Run workflow every 24 hours at 1am") to "Get Palo Alto security advisories".**

5. **Create Set node:**  
   - Name: "Extract info"  
   - Type: Set  
   - Add three string fields with expressions:  
     - `type`: `={{ $json.title.match(/[^ ]* ([^:]*):/)[1].trim() }}`  
     - `subject`: `={{ $json.title.match(/[^ ]* [^:]*: (.*)(?=\(Severity:)/)[1].trim() }}`  
     - `severity`: `={{ $json.title.split('Severity:')[1].replaceAll(')', '').trim().toLowerCase().toTitleCase() }}`  
   - Connect output of "Get Palo Alto security advisories" to "Extract info".

6. **Create If node:**  
   - Name: "Check if posted in last 24 hours"  
   - Type: If  
   - Condition: DateTime check to verify if `$json.pubDate` is newer than `{{$today.minus({days: 1})}}`  
   - Connect "Extract info" output to this node.

7. **Create No Operation node:**  
   - Name: "Ignore, stale advisory"  
   - Type: No Operation  
   - Connect false output of "Check if posted in last 24 hours" here.

8. **Create two Filter nodes:**  
   - Name: "GlobalProtect advisory?"  
     - Type: Filter  
     - Condition: `$json.title` contains "GlobalProtect".  
     - Connect true output of "Check if posted in last 24 hours" to this node.

   - Name: "Traps advisory?"  
     - Type: Filter  
     - Condition: `$json.title` contains "Traps".  
     - Connect true output of "Check if posted in last 24 hours" to this node.

9. **Create Jira node:**  
   - Name: "Create Jira issue"  
   - Type: Jira  
   - Configure credentials with Jira Software Cloud API credentials.  
   - Set Project and Issue Type (select appropriate project and issue type from your Jira instance).  
   - Summary: `={{ $json.title.substring(14) }}` (assumes fixed title prefix length).  
   - Description:  
     ```
     Severity: {{ $json.title.split('(Severity:')[1].replace(')', '').trim() }}
     Link: {{ $json.link }}
     Published: {{ $json.pubDate }}
     ```  
   - Priority: Set as desired.  
   - Connect output of "GlobalProtect advisory?" node to this Jira node.

10. **Create n8nTrainingCustomerDatastore node:**  
    - Name: "Get customers"  
    - Type: n8nTrainingCustomerDatastore (or replace with Google Sheets / company directory node)  
    - Operation: "getAllPeople"  
    - Return all records.  
    - Connect output of "Traps advisory?" node to this node.  
    - Also connect output of "Create Jira issue" node to this node (for downstream notifications).

11. **Create Gmail node:**  
    - Name: "Email customers"  
    - Type: Gmail (OAuth2)  
    - Configure with Gmail OAuth2 credentials.  
    - Send To: `={{ $json.email }}` (from customer data).  
    - Subject: `=New {{ $('Extract info').item.json.type }} security advisory`  
    - Message:  
      ```
      Dear {{ $json.name.split(' ')[0] }},

      We wanted to let you know of a new security advisory:

      {{ $('GlobalProtect advisory?').item.json.title }}
      {{ $('GlobalProtect advisory?').item.json.link }}

      Regards,

      Nathan
      ```  
    - Connect "Get customers" output to this node.

12. **Add Sticky Note nodes:**  
    - Add notes at appropriate positions in the canvas to explain workflow overview, RSS feed usage, deduplication, filtering, email notifications, and Jira integration as per the original workflow.

13. **Test and validate:**  
    - Verify Jira credentials and project setup.  
    - Replace the sample customer datastore node with a real data source if available.  
    - Confirm Gmail OAuth2 credentials are authorized to send emails.  
    - Run manually and observe logs for errors.  
    - Adjust schedule and date filters as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                   | Context or Link                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------|
| Workflow is designed to run daily at 1 AM; if scheduling changes, update the date filter in "Check if posted in last 24 hours" to match the new frequency (e.g., 7 days for weekly).                                                                                                                                                                    | Sticky Note5: Deduplication and scheduling                                |
| The sample customer datastore node is a placeholder; replace it with an actual company email directory integration such as Google Sheets or a corporate directory API. Expected JSON format includes `name` and `email` fields.                                                                                                                           | Sticky Note2: Customer directory guidance                                |
| The Gmail node can be substituted with other email providers by changing the node type and adjusting expressions accordingly.                                                                                                                                                                              | Sticky Note3: Email sending explanation                                   |
| To add new product filters, duplicate filter nodes and connect them after the date filter node. Connect to an incident management node or notification node as required.                                                                                                                                     | Sticky Note4: Adding filters and incident management                      |
| Jira integration relies on Jira Software Cloud API credentials; ensure credentials have appropriate permissions for issue creation and priority setting.                                                                                                                                                     | Jira node configuration                                                   |
| Expressions use JavaScript regex and string functions to parse advisory titles; changes in Palo Alto Networks RSS feed format may require adjustments to these expressions.                                                                                                                                  | Extract info node expressions                                            |
| The workflow includes multiple entry points (manual and scheduled triggers) for flexibility.                                                                                                                                                                                                                   | Workflow overview                                                         |
| Visual aids included in sticky notes highlight workflow design philosophy and node functions for easier maintenance and onboarding.                                                                                                                                                                           | Sticky notes with images and explanations                                |

---

This comprehensive analysis and reconstruction guide enables users and automation agents to understand, reproduce, and customize the "Monitor Security Advisories" n8n workflow efficiently while anticipating possible integration and operational issues.