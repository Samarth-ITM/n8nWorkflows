Automated Lead Capture, Scoring & CRM Integration with HubSpot, Clearbit & Slack

https://n8nworkflows.xyz/workflows/automated-lead-capture--scoring---crm-integration-with-hubspot--clearbit---slack-7343


# Automated Lead Capture, Scoring & CRM Integration with HubSpot, Clearbit & Slack

### 1. Workflow Overview

This workflow automates the capture, enrichment, scoring, and routing of sales leads from multiple sources. It is designed to intake raw lead data via a webhook, validate and enrich it using Clearbit and Apollo APIs, score leads based on company size, industry, role, and email quality, then route qualified leads to appropriate CRM actions and team notifications. The workflow is structured into the following logical blocks:

- **1.1 Lead Capture & Validation:** Accept incoming lead data via a webhook and validate essential fields, primarily the email.
- **1.2 Data Enrichment:** Use Clearbit and Apollo services to enrich lead data with company and person details.
- **1.3 Lead Scoring & Qualification:** Apply a scoring algorithm to qualify leads into High, Medium, Low, or Unqualified categories.
- **1.4 Routing & CRM Integration:** Depending on qualification, create contacts in HubSpot CRM and notify sales or marketing teams via Slack.
- **1.5 Error Handling & Logging:** Handle invalid data inputs and enrichment errors gracefully, ensuring data integrity and workflow stability.

This modular design enables clear separation of concerns, easy maintenance, and secure use of environment variables for sensitive credentials.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Capture & Validation

- **Overview:** This block captures lead data from external sources via a webhook, then validates that the lead contains a non-empty, correctly formatted email address before proceeding.
- **Nodes Involved:** Lead Capture Webhook, Validate Lead Data, Log Invalid Data, Sticky Notes (Webhook Setup, Data Validation, Error Handling).

- **Node Details:**

  - **Lead Capture Webhook**
    - Type: Webhook
    - Role: Entry point for inbound lead data via HTTP POST at endpoint `/lead-capture`.
    - Configuration: Expects POST requests with fields including email (required), firstName, lastName, company, phone, source, message.
    - Inputs: External HTTP requests.
    - Outputs: Data passed to validation node.
    - Edge Cases: Missing or malformed requests, wrong HTTP method.
  
  - **Validate Lead Data (If Node)**
    - Type: Conditional (If)
    - Role: Checks that the email field is present and matches a regex for valid email format.
    - Configuration:
      - Condition 1: Email is not empty.
      - Condition 2: Email matches regex pattern `^[\w-\.]+@[\w-]+\.[a-zA-Z]{2,}$`.
    - Inputs: Output from webhook.
    - Outputs:
      - True branch: Valid data → continues enrichment.
      - False branch: Invalid data → goes to logging.
    - Edge Cases: Emails with uncommon domains or characters might fail regex; empty or missing email.
  
  - **Log Invalid Data**
    - Type: No Operation (NoOp)
    - Role: Placeholder node to log or handle invalid lead data.
    - Inputs: Output of Validate Lead Data (False branch).
    - Outputs: None further.
    - Edge Cases: Could be extended to notify or reject requests.
  
  - **Sticky Notes**
    - Provide documentation on webhook setup, validation rules, and error handling strategies.
    - Emphasize required fields, data format, and error paths.

#### 2.2 Data Enrichment

- **Overview:** This block enriches validated leads by querying Clearbit and Apollo APIs to add company and personal data attributes, then merges enrichment results.
- **Nodes Involved:** Enrich with Clearbit, Enrich with Apollo (HTTP Request), Merge Enrichment Data, Sticky Notes (Enrichment, Environment Variables).

- **Node Details:**

  - **Enrich with Clearbit**
    - Type: Clearbit node (built-in)
    - Role: Fetches person-related enrichment data based on lead email.
    - Configuration: Uses email from lead data; resource type "person".
    - Inputs: Validated lead data.
    - Outputs: Enrichment data to merge node.
    - Requirements: Clearbit API credentials stored securely.
    - Edge Cases: API rate limits, missing data for some emails.

  - **Enrich with Apollo**
    - Type: HTTP Request
    - Role: Calls Apollo API to match person data using email, first name, and last name.
    - Configuration:
      - URL from environment variable `APOLLO_API_URL`.
      - HTTP POST with JSON body including email, first_name, last_name.
      - Header authentication via HTTP header (API Key).
    - Inputs: Validated lead data.
    - Outputs: Enrichment data to merge node.
    - Requirements: Apollo API credentials; environment variable setup.
    - Edge Cases: API downtime, authentication failures, partial results.
  
  - **Merge Enrichment Data**
    - Type: Merge
    - Role: Combines Clearbit and Apollo enrichment outputs into a single data object for scoring.
    - Inputs: Two main inputs from Clearbit and Apollo nodes.
    - Outputs: Consolidated enriched lead data.
    - Edge Cases: Conflicting data, missing fields.

  - **Sticky Notes**
    - Describe enrichment data fields (company size, industry, tech stack, social profiles, location).
    - Security note on API key usage.
    - Required environment variables and credential notes.

#### 2.3 Lead Scoring & Qualification

- **Overview:** Applies a custom scoring algorithm to leads enriched with company and personal data to classify leads into priority levels guiding sales follow-up.
- **Nodes Involved:** Calculate Lead Score (Code), Route by Qualification (If), Sticky Notes (Lead Scoring).

- **Node Details:**

  - **Calculate Lead Score**
    - Type: Code (JavaScript)
    - Role: Implements business logic to score leads based on company size, industry, job title, email domain, and revenue.
    - Configuration:
      - Scores assigned as per criteria:
        - Company size: 10 to 30 points.
        - Industry match: 25 points.
        - Title seniority: 15 to 25 points.
        - Business email domain: 10 points.
        - Revenue thresholds: 10 or 20 points.
      - Aggregates notes to explain scoring.
      - Determines qualification level and priority action.
    - Inputs: Merged enrichment data with original lead.
    - Outputs: Leads annotated with score, qualification level, priority, and notes.
    - Edge Cases: Missing enrichment metrics, null values, unexpected data types.

  - **Route by Qualification**
    - Type: Conditional (If)
    - Role: Branches leads based on score threshold ≥ 80.
    - Configuration:
      - True: High Priority path.
      - False: Medium or lower priority path.
    - Inputs: Output from scoring node.
    - Outputs: Branches for subsequent CRM/notification actions.
    - Edge Cases: Score exactly on boundary, missing score.

  - **Sticky Note**
    - Details qualification criteria for High, Medium, Low priority leads.

#### 2.4 Routing & CRM Integration

- **Overview:** Creates or updates contacts in HubSpot CRM based on lead priority and sends notifications to sales or marketing teams using Slack.
- **Nodes Involved:** Create HubSpot Contact (High Priority), Notify Sales Team - High Priority (Slack), Create HubSpot Contact - Medium, Notify Marketing - Medium Priority (Slack), Sticky Notes (High Priority Actions, Medium Priority Actions).

- **Node Details:**

  - **Create HubSpot Contact (High Priority)**
    - Type: HubSpot node
    - Role: Creates a new contact in HubSpot CRM for high-priority leads.
    - Configuration: Operation set to "create".
    - Inputs: High priority leads from routing node.
    - Outputs: HubSpot contact creation response.
    - Requirements: HubSpot API credentials.
    - Edge Cases: Duplicate contacts, API errors.
  
  - **Notify Sales Team - High Priority (Slack)**
    - Type: Slack node (Send Message)
    - Role: Sends alert message to designated sales Slack channel.
    - Configuration: Text alert with emoji, channel ID from environment variable `SLACK_SALES_CHANNEL_ID`.
    - Inputs: High priority leads.
    - Outputs: Slack notification result.
    - Requirements: Slack API credentials.
    - Edge Cases: Slack API downtime, invalid channel ID.
  
  - **Create HubSpot Contact - Medium**
    - Type: HubSpot node
    - Role: Creates contact for medium priority leads.
    - Configuration: Operation "create".
    - Inputs: Medium/low priority branch.
    - Outputs: HubSpot response.
    - Edge Cases: Same as high priority.
  
  - **Notify Marketing - Medium Priority (Slack)**
    - Type: Slack node
    - Role: Sends notification to marketing channel about medium priority leads.
    - Configuration: Channel ID from `SLACK_MARKETING_CHANNEL_ID`.
    - Inputs: Medium priority leads.
    - Outputs: Slack notification result.
    - Edge Cases: Same as above.
  
  - **Sticky Notes**
    - Describe actions per priority: high priority leads get immediate follow-up, notifications, and CRM creation; medium and low priorities get CRM creation and added to nurture sequences.

#### 2.5 Error Handling & Logging

- **Overview:** Defines how invalid lead data and enrichment failures are handled.
- **Nodes Involved:** Log Invalid Data, Sticky Note (Error Handling).

- **Node Details:**

  - **Log Invalid Data**
    - Role: Captures leads failing validation to prevent loss and enable monitoring.
  
  - **Sticky Note**
    - Specifies error handling paths:
      - Invalid data: log, notify, return 400.
      - Enrichment failures: proceed with base data, log errors, fallback scoring.
    - Emphasizes robustness and graceful degradation.

---

### 3. Summary Table

| Node Name                   | Node Type            | Functional Role                           | Input Node(s)             | Output Node(s)                     | Sticky Note                                                                                                                         |
|-----------------------------|----------------------|-----------------------------------------|---------------------------|----------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note - Overview       | Sticky Note          | Documentation overview                   |                           |                                  | ## Lead Capture & Auto-Qualification Workflow - Purpose, Security, Flow summary                                                   |
| Lead Capture Webhook         | Webhook              | Entry point for inbound leads            |                           | Validate Lead Data               | ## Webhook Configuration - Endpoint, method, expected data                                                                         |
| Sticky Note - Webhook Setup  | Sticky Note          | Webhook setup documentation              |                           |                                  | ## Webhook Configuration - Endpoint, method, expected data                                                                         |
| Validate Lead Data           | If                   | Validate presence and format of email   | Lead Capture Webhook       | Enrich with Clearbit, Enrich with Apollo, Log Invalid Data | ## Data Validation Rules - Required fields, optional enhancements                                                                   |
| Log Invalid Data             | NoOp                 | Handles invalid lead data                | Validate Lead Data (False) |                                  | ## Error Handling - Invalid data path logging                                                                                      |
| Enrich with Clearbit         | Clearbit             | Enrich lead data with Clearbit API      | Validate Lead Data (True)  | Merge Enrichment Data            | ## Lead Enrichment - Clearbit data, security note                                                                                  |
| Enrich with Apollo           | HTTP Request         | Enrich lead data with Apollo API        | Validate Lead Data (True)  | Merge Enrichment Data            | ## Lead Enrichment - Clearbit data, security note                                                                                  |
| Merge Enrichment Data        | Merge                | Combine enrichment results               | Enrich with Clearbit, Enrich with Apollo | Calculate Lead Score     |                                                                                                                                     |
| Calculate Lead Score         | Code                 | Score and qualify lead                   | Merge Enrichment Data      | Route by Qualification          | ## Lead Qualification Criteria - Scoring thresholds and priority levels                                                           |
| Route by Qualification       | If                   | Branch leads by score                    | Calculate Lead Score       | Create HubSpot Contact, Notify Sales Team - High Priority, Create HubSpot Contact - Medium, Notify Marketing - Medium Priority |                                                                                                                                     |
| Create HubSpot Contact       | HubSpot              | Create contact for high priority leads  | Route by Qualification    (True) | Notify Sales Team - High Priority | ## High Priority Lead Actions - CRM creation, notifications, follow-ups                                                            |
| Notify Sales Team - High Priority | Slack            | Notify sales team on high priority leads| Create HubSpot Contact     |                                  | ## High Priority Lead Actions - CRM creation, notifications, follow-ups                                                            |
| Create HubSpot Contact - Medium | HubSpot           | Create contact for medium priority leads| Route by Qualification (False) | Notify Marketing - Medium Priority | ## Medium/Low Priority Actions - CRM creation, nurture sequences                                                                    |
| Notify Marketing - Medium Priority | Slack           | Notify marketing on medium priority leads| Create HubSpot Contact - Medium |                              | ## Medium/Low Priority Actions - CRM creation, nurture sequences                                                                    |
| Sticky Note - Data Validation | Sticky Note         | Validation rules documentation          |                           |                                  | ## Data Validation Rules - Required fields, optional enhancements                                                                   |
| Sticky Note - Enrichment     | Sticky Note          | Enrichment overview                      |                           |                                  | ## Lead Enrichment - Clearbit data, security note                                                                                  |
| Sticky Note - Lead Scoring   | Sticky Note          | Lead scoring criteria documentation     |                           |                                  | ## Lead Qualification Criteria - Scoring thresholds and priority levels                                                           |
| Sticky Note - High Priority Actions | Sticky Note    | High priority lead actions overview     |                           |                                  | ## High Priority Lead Actions - CRM creation, notifications, follow-ups                                                            |
| Sticky Note - Medium Priority Actions | Sticky Note   | Medium/low priority lead actions overview|                           |                                  | ## Medium/Low Priority Actions - CRM creation, nurture sequences                                                                    |
| Sticky Note - Error Handling | Sticky Note          | Error handling documentation             |                           |                                  | ## Error Handling - Invalid data and enrichment failure strategies                                                                 |
| Sticky Note - Environment Variables | Sticky Note     | Required environment variables and credentials |                      |                                  | ## Required Environment Variables - API URLs, Slack channels, credentials                                                         |
| Sticky Note - Integration Points | Sticky Note        | Explanation of sources feeding the webhook |                         |                                  | ## Integration Sources - Possible lead origin channels and POST format                                                             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**
   - Type: Webhook
   - Name: Lead Capture Webhook
   - HTTP Method: POST
   - Path: `lead-capture`
   - No authentication
   - Save and activate.

2. **Create If Node for Validation:**
   - Type: If
   - Name: Validate Lead Data
   - Conditions:
     - Email field is not empty: `{{$json.email}} != ''`
     - Email matches regex: `{{$json.email}}` matches `^[\w-\.]+@[\w-]+\.[a-zA-Z]{2,}$`
   - Connect webhook main output to this node input.

3. **Create NoOp Node for Invalid Data Logging:**
   - Type: NoOp
   - Name: Log Invalid Data
   - Connect Validate Lead Data False output here.

4. **Create Clearbit Node for Enrichment:**
   - Type: Clearbit
   - Name: Enrich with Clearbit
   - Resource: Person
   - Email: `{{$json.email}}`
   - Credentials: Configure Clearbit API credential.
   - Connect Validate Lead Data True output here.

5. **Create HTTP Request Node for Apollo Enrichment:**
   - Type: HTTP Request
   - Name: Enrich with Apollo
   - HTTP Method: POST
   - URL: `{{$env.APOLLO_API_URL}}/v1/people/match`
   - Authentication: HTTP Header Auth with API key stored in credentials
   - Headers: Content-Type `application/json`, Cache-Control `no-cache`
   - Body (JSON):
     ```json
     {
       "email": "{{$json.email}}",
       "first_name": "{{$json.firstName}}",
       "last_name": "{{$json.lastName}}"
     }
     ```
   - Connect Validate Lead Data True output here.

6. **Create Merge Node:**
   - Type: Merge
   - Name: Merge Enrichment Data
   - Mode: Append (default)
   - Connect outputs from Clearbit and Apollo enrichment nodes to inputs 0 and 1 respectively.
   
7. **Create Code Node for Lead Scoring:**
   - Type: Code
   - Name: Calculate Lead Score
   - Language: JavaScript
   - Paste scoring logic as described in node details.
   - Input: merged enrichment data.
   - Output: enriched and scored lead.

8. **Create If Node for Routing:**
   - Type: If
   - Name: Route by Qualification
   - Condition: `$json.qualification.score >= 80`
   - Connect output of scoring node to this node.

9. **Create HubSpot Node for High Priority Contact:**
   - Type: HubSpot
   - Name: Create HubSpot Contact
   - Operation: Create
   - Credentials: HubSpot API credentials
   - Connect Route by Qualification True output here.

10. **Create Slack Node for High Priority Notification:**
    - Type: Slack
    - Name: Notify Sales Team - High Priority
    - Message text: "🚨 **High Priority Lead Alert!** 🚨"
    - Channel: Use environment variable `SLACK_SALES_CHANNEL_ID`
    - Credentials: Slack API credentials
    - Connect output of HubSpot create node here.

11. **Create HubSpot Node for Medium Priority Contact:**
    - Type: HubSpot
    - Name: Create HubSpot Contact - Medium
    - Operation: Create
    - Connect Route by Qualification False output here.

12. **Create Slack Node for Medium Priority Notification:**
    - Type: Slack
    - Name: Notify Marketing - Medium Priority
    - Message text: "📋 New Medium Priority Lead"
    - Channel: Use environment variable `SLACK_MARKETING_CHANNEL_ID`
    - Connect output of HubSpot medium create node here.

13. **Add Sticky Notes:**
    - Create sticky notes for overview, webhook setup, data validation, enrichment, lead scoring, priority action descriptions, error handling, environment variables, integration points as per content described.

14. **Configure Credentials and Environment Variables:**
    - Clearbit API key credential.
    - Apollo API key credential with HTTP Header Auth.
    - HubSpot API credentials.
    - Slack API credentials.
    - Environment variables: `APOLLO_API_URL`, `SLACK_SALES_CHANNEL_ID`, `SLACK_MARKETING_CHANNEL_ID`, `CRM_ASSIGNMENT_URL`.

15. **Test the workflow:**
    - Send test POST requests with various lead data.
    - Verify validation, enrichment, scoring, routing, CRM record creation, and Slack notifications.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| Workflow uses environment variables exclusively for API keys to enhance security; no hardcoded secrets.   | Security best practice                                                                                    |
| Integrates multiple lead sources including web forms, LinkedIn, Facebook, webinars, and content downloads. | Integration Points sticky note                                                                         |
| Lead scoring logic is customizable in the code node to fit different business criteria and markets.       | Lead Scoring sticky note                                                                               |
| Slack notifications improve sales and marketing team responsiveness for lead follow-up.                   | Slack nodes and priority action sticky notes                                                         |
| HubSpot CRM integration facilitates automated contact creation and lead management.                       | HubSpot nodes                                                                                        |
| For further info on Clearbit API and Apollo API usage, consult their official documentation.              | External APIs                                                                                         |
| Error handling includes logging invalid inputs and fallback scoring to avoid workflow failures.           | Error Handling sticky note                                                                             |

---

This documentation is derived exclusively from the provided n8n workflow JSON and offers a comprehensive, precise, and actionable reference for developers and automation agents.