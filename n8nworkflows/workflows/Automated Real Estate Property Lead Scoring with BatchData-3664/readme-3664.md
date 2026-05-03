Automated Real Estate Property Lead Scoring with BatchData

https://n8nworkflows.xyz/workflows/automated-real-estate-property-lead-scoring-with-batchdata-3664


# Automated Real Estate Property Lead Scoring with BatchData

### 1. Workflow Overview

This workflow automates the qualification and scoring of real estate leads by integrating CRM lead data with property information from BatchData’s API. It enriches leads with verified property details, applies a multi-factor scoring algorithm, updates the CRM with qualification results, and routes high-value leads for immediate follow-up and notifications.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures new lead data via a webhook from the CRM.
- **1.2 Lead Data Retrieval:** Fetches comprehensive lead details from the CRM API.
- **1.3 Property Data Enrichment:** Queries BatchData API to obtain detailed property information.
- **1.4 Lead Scoring and Qualification:** Applies a scoring algorithm based on property characteristics and classifies the lead.
- **1.5 CRM Update:** Updates the CRM record with enriched data and qualification results.
- **1.6 Lead Routing:** Routes leads based on qualification status, triggering follow-up tasks for high-value leads.
- **1.7 Notifications:** Sends alerts (Slack in this example) about high-value leads.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives new lead notifications from the CRM via a webhook, capturing essential address and lead identifiers to initiate the qualification process.

**Nodes Involved:**  
- CRM New Lead Webhook  
- Webhook Setup Instructions (Sticky Note)

**Node Details:**

- **CRM New Lead Webhook**  
  - *Type:* Webhook  
  - *Role:* Entry point for new lead data from CRM systems.  
  - *Configuration:* Listens on path `/crm-new-lead` for incoming HTTP POST requests.  
  - *Key Expressions:* None (raw payload expected).  
  - *Inputs:* External CRM webhook calls.  
  - *Outputs:* Passes JSON payload containing `leadId`, `crmApiUrl`, `address`, `city`, `state`, and `zipcode` to next node.  
  - *Edge Cases:* Missing or malformed payload fields; webhook URL misconfiguration; unauthorized requests.  
  - *Sticky Note:* Instructions specify expected payload format and mandatory fields.

- **Webhook Setup Instructions**  
  - *Type:* Sticky Note  
  - *Role:* Provides setup guidance for configuring the CRM webhook and expected data format.  
  - *Content Highlights:* Payload must include `leadId`, `crmApiUrl`, and full address details; all fields required for property verification.

---

#### 2.2 Lead Data Retrieval

**Overview:**  
Fetches detailed lead information from the CRM API using the lead ID obtained from the webhook, ensuring the workflow has complete and accurate lead data.

**Nodes Involved:**  
- Fetch Lead Data  
- CRM API Instructions (Sticky Note)

**Node Details:**

- **Fetch Lead Data**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves full lead record from CRM API.  
  - *Configuration:*  
    - URL constructed dynamically as `{{ $json.crmApiUrl }}/leads/{{ $json.leadId }}`.  
    - Authentication via HTTP Header Auth credentials (configured separately).  
  - *Key Expressions:* URL parameters use incoming JSON fields.  
  - *Inputs:* Output from webhook node.  
  - *Outputs:* Lead data JSON passed downstream.  
  - *Edge Cases:* API authentication failure; lead not found; network timeouts; invalid URL.  
  - *Sticky Note:* Instructions for setting up CRM API credentials and ensuring address data is included in response.

---

#### 2.3 Property Data Enrichment

**Overview:**  
Queries BatchData’s property API with the lead’s address to retrieve detailed property information necessary for scoring.

**Nodes Involved:**  
- BatchData Property Lookup  
- BatchData API Instructions (Sticky Note)

**Node Details:**

- **BatchData Property Lookup**  
  - *Type:* HTTP Request  
  - *Role:* Calls BatchData API to fetch property details.  
  - *Configuration:*  
    - POST request to `https://api.batchdata.com/api/v1/property/search`.  
    - Body includes address, city, state, and zipcode from lead data.  
    - Authentication via HTTP Header Auth with BatchData API key.  
  - *Key Expressions:* Body parameters dynamically populated from lead JSON.  
  - *Inputs:* Lead data from previous node.  
  - *Outputs:* Property data JSON passed to scoring node.  
  - *Edge Cases:* API key invalid or expired; property not found; API rate limits; malformed address data.  
  - *Sticky Note:* Setup instructions for BatchData account, API key, and expected response fields.

---

#### 2.4 Lead Scoring and Qualification

**Overview:**  
Executes a JavaScript function to score the lead based on property characteristics and classify it into qualification categories.

**Nodes Involved:**  
- Score And Qualify Lead  
- Lead Scoring Instructions (Sticky Note)

**Node Details:**

- **Score And Qualify Lead**  
  - *Type:* Code (JavaScript)  
  - *Role:* Implements scoring algorithm and lead qualification logic.  
  - *Configuration:*  
    - Scores based on property value, square footage, age, investment status, and lot size.  
    - Assigns points per criteria and aggregates total score.  
    - Classifies leads as `high-value`, `qualified`, `potential`, `not qualified`, or `unverified`.  
    - Returns enriched data object including score, status, notes, and detailed property info.  
  - *Key Expressions:* Uses current year for age calculation; conditional scoring thresholds.  
  - *Inputs:* Property data JSON from BatchData API.  
  - *Outputs:* Enriched lead data with scoring results.  
  - *Edge Cases:* Missing or incomplete property data; unexpected data types; zero or null values.  
  - *Sticky Note:* Detailed explanation of scoring factors and thresholds for customization.

---

#### 2.5 CRM Update

**Overview:**  
Updates the CRM lead record with the enriched property data, scoring results, and qualification status.

**Nodes Involved:**  
- Update CRM Lead  
- CRM Update Instructions (Sticky Note)

**Node Details:**

- **Update CRM Lead**  
  - *Type:* HTTP Request  
  - *Role:* Sends PUT request to CRM API to update lead record.  
  - *Configuration:*  
    - URL: `{{ $json.crmApiUrl }}/leads/{{ $json.leadId }}`.  
    - Method: PUT (adjustable).  
    - Body parameters include score, qualification status, notes, property value, size, beds, baths, and verification flag.  
    - Authentication: HTTP Header Auth credentials.  
  - *Key Expressions:* Dynamic field mapping from enriched data JSON.  
  - *Inputs:* Output from scoring node.  
  - *Outputs:* Passes updated lead data to routing logic.  
  - *Edge Cases:* API write permission issues; field name mismatches; network errors; partial updates.  
  - *Sticky Note:* Guidance on adjusting field names and HTTP method per CRM API requirements.

---

#### 2.6 Lead Routing

**Overview:**  
Determines if a lead is high-value and routes it accordingly for immediate follow-up or standard notification.

**Nodes Involved:**  
- Is High-Value Lead? (If node)  
- Routing Instructions (Sticky Note)

**Node Details:**

- **Is High-Value Lead?**  
  - *Type:* If (Conditional)  
  - *Role:* Checks if `qualificationStatus` equals `high-value`.  
  - *Configuration:* String condition comparing `{{$json.qualificationStatus}}` to `"high-value"`.  
  - *Inputs:* Updated lead data from CRM update node.  
  - *Outputs:*  
    - True path: High-value leads.  
    - False path: All other leads.  
  - *Edge Cases:* Missing `qualificationStatus`; unexpected values.  
  - *Sticky Note:* Explains routing logic and suggests customization options for conditions.

---

#### 2.7 Notifications and Follow-up Tasks

**Overview:**  
Creates an immediate follow-up task for high-value leads and sends Slack notifications to alert the sales team.

**Nodes Involved:**  
- Create Immediate Follow-up Task  
- Follow-up Task Instructions (Sticky Note)  
- Send Slack Notification  
- Notification Instructions (Sticky Note)

**Node Details:**

- **Create Immediate Follow-up Task**  
  - *Type:* HTTP Request  
  - *Role:* Creates a high-priority follow-up task in the CRM system.  
  - *Configuration:*  
    - POST to `{{ $json.crmApiUrl }}/tasks`.  
    - Body includes task type, lead ID, priority, due date (today), note with property value, and assigned user (`senior-agent`).  
    - Authentication: HTTP Header Auth.  
  - *Inputs:* True output from "Is High-Value Lead?" node.  
  - *Outputs:* None (terminal for this path).  
  - *Edge Cases:* Task creation failure; invalid assignee; date formatting issues.  
  - *Sticky Note:* Instructions for customizing task parameters and alternative alerting methods.

- **Send Slack Notification**  
  - *Type:* Slack  
  - *Role:* Sends a notification message to a Slack channel about the high-value lead.  
  - *Configuration:*  
    - Channel ID: `high-value-leads`.  
    - Message includes lead ID, property value, score, and qualification notes.  
  - *Inputs:* False output from "Is High-Value Lead?" node (standard leads).  
  - *Outputs:* None (terminal).  
  - *Edge Cases:* Slack credential misconfiguration; invalid channel ID; message formatting errors.  
  - *Sticky Note:* Guidance on Slack credential setup, channel customization, and alternative notification options.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                          | Input Node(s)               | Output Node(s)                     | Sticky Note                                                                                          |
|-------------------------------|---------------------|----------------------------------------|-----------------------------|----------------------------------|----------------------------------------------------------------------------------------------------|
| CRM New Lead Webhook           | Webhook             | Receives new lead data from CRM        | External webhook call       | Fetch Lead Data                   | # WEBHOOK SETUP INSTRUCTIONS: Payload format and required fields                                  |
| Webhook Setup Instructions    | Sticky Note         | Setup guidance for webhook              | None                       | None                             | # WEBHOOK SETUP INSTRUCTIONS: Payload format and required fields                                  |
| Fetch Lead Data                | HTTP Request        | Fetches full lead data from CRM API    | CRM New Lead Webhook        | BatchData Property Lookup         | # CRM API CONFIGURATION: Setup HTTP Header Auth credentials and ensure address data                |
| CRM API Instructions          | Sticky Note         | CRM API credential and usage guidance  | None                       | None                             | # CRM API CONFIGURATION: Setup HTTP Header Auth credentials and ensure address data                |
| BatchData Property Lookup      | HTTP Request        | Queries BatchData API for property info| Fetch Lead Data             | Score And Qualify Lead            | # BATCHDATA API SETUP: API key setup and expected response                                        |
| BatchData API Instructions    | Sticky Note         | BatchData API setup instructions       | None                       | None                             | # BATCHDATA API SETUP: API key setup and expected response                                        |
| Score And Qualify Lead         | Code (JavaScript)   | Scores and classifies the lead          | BatchData Property Lookup   | Update CRM Lead                  | # LEAD SCORING ALGORITHM: Detailed scoring factors and thresholds                                 |
| Lead Scoring Instructions     | Sticky Note         | Explains scoring logic and thresholds  | None                       | None                             | # LEAD SCORING ALGORITHM: Detailed scoring factors and thresholds                                 |
| Update CRM Lead               | HTTP Request        | Updates CRM with enriched lead data    | Score And Qualify Lead      | Is High-Value Lead?              | # CRM UPDATE CONFIGURATION: Adjust field mappings and HTTP method                                 |
| CRM Update Instructions       | Sticky Note         | CRM update configuration guidance      | None                       | None                             | # CRM UPDATE CONFIGURATION: Adjust field mappings and HTTP method                                 |
| Is High-Value Lead?            | If                  | Routes leads based on qualification    | Update CRM Lead             | Create Immediate Follow-up Task (True), Send Slack Notification (False) | # ROUTING LOGIC: Conditional routing based on qualification status                                |
| Routing Instructions          | Sticky Note         | Explains routing logic and customization| None                       | None                             | # ROUTING LOGIC: Conditional routing based on qualification status                                |
| Create Immediate Follow-up Task| HTTP Request        | Creates urgent follow-up task in CRM   | Is High-Value Lead? (True)  | None                             | # HIGH-VALUE LEAD HANDLING: Task creation and alternative alerting methods                        |
| Follow-up Task Instructions   | Sticky Note         | Guidance on follow-up task parameters   | None                       | None                             | # HIGH-VALUE LEAD HANDLING: Task creation and alternative alerting methods                        |
| Send Slack Notification       | Slack               | Sends notification about lead          | Is High-Value Lead? (False) | None                             | # NOTIFICATION CONFIGURATION: Slack setup and alternative notification options                    |
| Notification Instructions     | Sticky Note         | Slack notification configuration guidance| None                       | None                             | # NOTIFICATION CONFIGURATION: Slack setup and alternative notification options                    |
| Workflow Overview             | Sticky Note         | High-level workflow summary             | None                       | None                             | Overview and setup checklist for the entire workflow                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - Name: `CRM New Lead Webhook`  
   - Path: `crm-new-lead`  
   - Purpose: Receive new lead data from CRM webhook.  
   - No authentication required.  

2. **Add Sticky Note:**  
   - Content: Webhook setup instructions with expected JSON payload including `leadId`, `crmApiUrl`, `address`, `city`, `state`, `zipcode`.  

3. **Create HTTP Request Node:**  
   - Name: `Fetch Lead Data`  
   - Method: GET  
   - URL: `={{ $json.crmApiUrl }}/leads/{{ $json.leadId }}`  
   - Authentication: HTTP Header Auth credentials for CRM API (set up separately).  
   - Purpose: Retrieve detailed lead data including address.  
   - Connect output of webhook node to this node.  

4. **Add Sticky Note:**  
   - Content: CRM API configuration instructions including credential setup and required data fields.  

5. **Create HTTP Request Node:**  
   - Name: `BatchData Property Lookup`  
   - Method: POST  
   - URL: `https://api.batchdata.com/api/v1/property/search`  
   - Authentication: HTTP Header Auth credentials with BatchData API key.  
   - Body Parameters (JSON):  
     ```json
     {
       "address": "={{ $json.address }}",
       "city": "={{ $json.city }}",
       "state": "={{ $json.state }}",
       "zipcode": "={{ $json.zipcode }}"
     }
     ```  
   - Connect output of `Fetch Lead Data` node to this node.  

6. **Add Sticky Note:**  
   - Content: BatchData API setup instructions including API key creation and expected response fields.  

7. **Create Code Node:**  
   - Name: `Score And Qualify Lead`  
   - Language: JavaScript  
   - Paste the scoring algorithm code that:  
     - Initializes score and notes  
     - Scores property value, size, age, investment status, lot size  
     - Assigns qualification status based on total score  
     - Returns enriched lead data object  
   - Connect output of `BatchData Property Lookup` node to this node.  

8. **Add Sticky Note:**  
   - Content: Detailed explanation of the lead scoring algorithm and thresholds.  

9. **Create HTTP Request Node:**  
   - Name: `Update CRM Lead`  
   - Method: PUT (or PATCH if your CRM requires)  
   - URL: `={{ $json.crmApiUrl }}/leads/{{ $json.leadId }}`  
   - Authentication: HTTP Header Auth credentials for CRM API.  
   - Body Parameters: Include fields for score, qualificationStatus, qualificationNotes, propertyValue, squareFootage, yearBuilt, bedrooms, bathrooms, batchDataVerified (boolean).  
   - Connect output of `Score And Qualify Lead` node to this node.  

10. **Add Sticky Note:**  
    - Content: CRM update instructions including field mapping and HTTP method considerations.  

11. **Create If Node:**  
    - Name: `Is High-Value Lead?`  
    - Condition: String equals  
      - Value 1: `={{ $json.qualificationStatus }}`  
      - Value 2: `high-value`  
    - Connect output of `Update CRM Lead` node to this node.  

12. **Add Sticky Note:**  
    - Content: Routing logic explanation and customization options.  

13. **Create HTTP Request Node:**  
    - Name: `Create Immediate Follow-up Task`  
    - Method: POST  
    - URL: `={{ $json.crmApiUrl }}/tasks`  
    - Authentication: HTTP Header Auth credentials for CRM API.  
    - Body Parameters:  
      - `type`: `immediate-followup`  
      - `leadId`: `={{ $json.leadId }}`  
      - `priority`: `high`  
      - `dueDate`: `={{ $now.format("YYYY-MM-DD") }}`  
      - `note`: `High-value lead with property value of ${{ $json.propertyData.estimatedValue }}. Immediate follow-up required.`  
      - `assignedTo`: `senior-agent`  
    - Connect True output of `Is High-Value Lead?` node here.  

14. **Add Sticky Note:**  
    - Content: Follow-up task creation instructions and alternative alerting options.  

15. **Create Slack Node:**  
    - Name: `Send Slack Notification`  
    - Channel ID: `high-value-leads` (adjust to your workspace)  
    - Message Text:  
      ```
      High-value lead alert: {{ $json.leadId }}
      Property Value: ${{ $json.propertyData.estimatedValue }}
      Score: {{ $json.score }}
      Qualification Notes: {{ $json.qualificationNotes }}
      ```  
    - Connect False output of `Is High-Value Lead?` node here.  

16. **Add Sticky Note:**  
    - Content: Slack notification setup instructions and alternative notification methods.  

17. **Create Sticky Note:**  
    - Name: `Workflow Overview`  
    - Content: Summary of workflow purpose, setup checklist, and key integration points.  

18. **Credential Setup:**  
    - Configure HTTP Header Auth credentials for:  
      - CRM API (with appropriate authorization headers)  
      - BatchData API (with `x-api-key` header)  
    - Configure Slack credentials in n8n Credentials Manager.  

19. **Test Workflow:**  
    - Use sample lead data matching the webhook payload format.  
    - Verify each step completes successfully and data flows correctly.  
    - Adjust scoring thresholds and API field mappings as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                                                                                  |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| BatchData provides comprehensive US property data including valuation, ownership, and tax information.         | https://batchdata.com/                                                                           |
| Scoring algorithm is customizable; adjust points and thresholds to fit your business needs and market.         | See Lead Scoring Instructions sticky note in workflow                                          |
| Slack notifications can be replaced with email, SMS, Microsoft Teams, or other communication nodes.            | Notification Instructions sticky note                                                          |
| CRM API endpoints and field names must be adapted to your specific CRM system’s API schema and authentication. | CRM API Instructions and CRM Update Instructions sticky notes                                  |
| The workflow executes within seconds of lead receipt, enabling real-time prioritization and routing.            | Workflow Overview sticky note                                                                  |

---

This documentation provides a detailed, structured reference for understanding, reproducing, and customizing the Automated Real Estate Property Lead Scoring workflow integrating BatchData with your CRM and notification systems.