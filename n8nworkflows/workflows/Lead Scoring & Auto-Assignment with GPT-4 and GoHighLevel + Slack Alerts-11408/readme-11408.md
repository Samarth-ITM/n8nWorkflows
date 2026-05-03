Lead Scoring & Auto-Assignment with GPT-4 and GoHighLevel + Slack Alerts

https://n8nworkflows.xyz/workflows/lead-scoring---auto-assignment-with-gpt-4-and-gohighlevel---slack-alerts-11408


# Lead Scoring & Auto-Assignment with GPT-4 and GoHighLevel + Slack Alerts

### 1. Workflow Overview

This workflow automates lead scoring and assignment for new contacts created in GoHighLevel (GHL), leveraging GPT-4 AI to analyze lead data and determine lead quality. It integrates with GHL to fetch detailed contact and engagement data, uses AI to generate a numeric lead score and category (Hot, Warm, Cold), tags and assigns the lead to appropriate sales team members, and sends Slack alerts for high-priority leads.

Logical blocks:

- **1.1 Input Reception & Configuration**  
  Receives new contact notifications from GHL via webhook and loads essential configuration parameters such as API keys, user IDs, and Slack channel.

- **1.2 Data Retrieval & Preparation**  
  Fetches full contact details and engagement data from GHL, then cleans and structures the data into a concise JSON object for AI processing.

- **1.3 AI Lead Scoring**  
  Sends the prepared lead data to GPT-4 AI for scoring and classification. Parses AI output to extract numeric score, category, confidence, and reasoning.

- **1.4 Lead Routing & Assignment**  
  Based on the AI score, routes leads into Hot, Warm, or Cold categories. Tags contacts accordingly in GHL, assigns them to the correct sales rep or team, and triggers Slack notifications for Hot leads.

- **1.5 Webhook Response**  
  Sends a final JSON response back to the GHL webhook caller confirming successful scoring and assignment.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & Configuration

- **Overview:**  
  Listens for new contact creation events from GoHighLevel via a webhook trigger and sets up workflow configuration variables including API credentials, user IDs for assignment, and Slack channel info.

- **Nodes Involved:**  
  - GHL New Contact Trigger  
  - Workflow Configuration  
  - Sticky Note1

- **Node Details:**  
  - **GHL New Contact Trigger**  
    - Type: Webhook Trigger  
    - Config: Listens on POST `/ghl-new-contact` path  
    - Role: Entry point receiving new contact data from GHL  
    - Inputs: External HTTP POST from GHL webhook  
    - Outputs: Passes contact ID and payload downstream  
    - Edge Cases: Missing or malformed webhook payload, unauthorized requests if webhook security is not configured  
  - **Workflow Configuration**  
    - Type: Set node  
    - Config: Stores static or environment-specific values such as:  
      - GoHighLevel API Base URL  
      - GoHighLevel API Key  
      - User IDs for top, secondary reps, nurture team  
      - Slack channel ID for alerts  
    - Role: Centralizes configuration for use by downstream API calls and notifications  
    - Inputs: From webhook trigger  
    - Outputs: Supplies config JSON for HTTP requests and Slack node  
    - Edge Cases: Missing or invalid configuration values will cause HTTP requests to fail or misroute leads  
  - **Sticky Note1**  
    - Provides overview documentation for this block

---

#### 1.2 Data Retrieval & Preparation

- **Overview:**  
  Retrieves full contact details and engagement activities from GHL using their API. Cleans and consolidates this data into a structured object tailored for AI consumption.

- **Nodes Involved:**  
  - GHL Get Contact Details  
  - GHL Fetch Engagement Data  
  - Clean Contact Data  
  - Prepare AI Input  
  - Sticky Note2

- **Node Details:**  
  - **GHL Get Contact Details**  
    - Type: HTTP Request  
    - Config: GET request to `/contacts/{contact_id}` endpoint of GHL API  
    - Headers: Authorization Bearer token with API Key from config  
    - Inputs: contact_id from webhook  
    - Outputs: JSON contact details including name, email, phone, tags, custom fields  
    - Edge Cases: API errors (401 unauthorized, 404 not found), rate limiting, missing contact ID  
  - **GHL Fetch Engagement Data**  
    - Type: HTTP Request  
    - Config: GET request to `/contacts/{contact_id}/activities` endpoint  
    - Headers: Authorization Bearer token  
    - Inputs: contact_id from webhook  
    - Outputs: Engagement activity history JSON array  
    - Edge Cases: API failures, empty engagement data, network timeouts  
  - **Clean Contact Data**  
    - Type: Set node  
    - Config: Extracts and normalizes key fields from the combined contact details and engagement data, ensures fallback defaults for missing values  
    - Variables set: contactId, name, email, phone, source, tags, customFields, engagementData  
    - Inputs: Outputs from previous two HTTP requests  
    - Outputs: Cleaned JSON for AI input  
  - **Prepare AI Input**  
    - Type: Set node  
    - Config: Organizes cleaned contact data into an object separating demographics and behavioral data, optimized for prompt consumption by AI lead scoring node  
    - Inputs: Cleaned contact data  
    - Outputs: JSON object with "leadData" property containing structured lead info  
  - **Sticky Note2**  
    - Documents this data preparation step

---

#### 1.3 AI Lead Scoring

- **Overview:**  
  Uses GPT-4 model via OpenAI integration to analyze lead data and assign a numeric score (1-100), category (Hot/Warm/Cold), confidence level, and brief reasoning. Parses the AI response to extract structured data for routing logic.

- **Nodes Involved:**  
  - AI Lead Scoring (GPT-4)  
  - Parse Score to Numeric  
  - Sticky Note3

- **Node Details:**  
  - **AI Lead Scoring (GPT-4)**  
    - Type: Langchain OpenAI node  
    - Config:  
      - Model: GPT-4 (model ID: gpt-4o)  
      - Instructions prompt detailing scoring criteria: buying intent, engagement, behavior, lead quality, urgency  
      - Expects JSON output with fields: score, category, reasoning, confidence  
      - Input: JSON stringified leadData from previous node  
    - Credentials: OpenAI API key required  
    - Outputs: Raw AI textual response with scoring JSON or text  
    - Edge Cases: AI response parsing errors, API rate limits, incomplete JSON in response  
  - **Parse Score to Numeric**  
    - Type: Code node (JavaScript)  
    - Config:  
      - Parses AI response JSON string safely, fallback to regex extraction if parsing fails  
      - Normalizes score between 1 and 100  
      - Adds contact metadata for downstream use  
    - Inputs: AI node output  
    - Outputs: JSON with numeric score, category, confidence, reasoning, and contact info  
    - Edge Cases: Parsing failures, missing fields default to medium confidence and score 50  
  - **Sticky Note3**  
    - Explains AI scoring and parsing logic

---

#### 1.4 Lead Routing & Assignment

- **Overview:**  
  Applies conditional logic to route leads based on score thresholds ≥80 (Hot), ≥40 (Warm), else Cold. Tags the lead in GHL accordingly, assigns to specific users (top rep, secondary rep, nurture team), and sends Slack alert for Hot leads.

- **Nodes Involved:**  
  - IF Score >= 80 (Hot)  
  - IF Score >= 40 (Warm)  
  - GHL Tag Hot Lead  
  - GHL Tag Warm Lead  
  - GHL Tag Cold Lead  
  - GHL Assign Hot to Top Rep  
  - GHL Assign Warm to Secondary Rep  
  - GHL Assign Cold to Nurture Team  
  - Slack Notify Hot Lead  
  - Sticky Note4

- **Node Details:**  
  - **IF Score >= 80 (Hot)**  
    - Type: If node  
    - Condition: score field exists and score ≥ 80  
    - True branch: Hot lead handling  
    - False branch: passes to Warm lead check  
  - **IF Score >= 40 (Warm)**  
    - Type: If node  
    - Condition: score field exists and score ≥ 40  
    - True branch: Warm lead handling  
    - False branch: Cold lead handling  
  - **GHL Tag Hot Lead**  
    - Type: HTTP Request (PUT)  
    - Endpoint: `/contacts/{contactId}`  
    - Body: sets tag "Hot Lead"  
    - Headers: Authorization Bearer, Content-Type JSON  
    - Inputs: contactId from parsed AI score  
  - **GHL Assign Hot to Top Rep**  
    - Type: HTTP Request (PUT)  
    - Endpoint: `/contacts/{contactId}/assign`  
    - Body: assigns to topRepUserId from config  
    - Followed by Slack notification node  
  - **Slack Notify Hot Lead**  
    - Type: Slack node  
    - Sends alert message to configured Slack channel with lead details and reasoning  
    - Credentials: Slack API OAuth2  
  - **GHL Tag Warm Lead**  
    - Similar to Hot Lead tag but with "Warm Lead" tag  
  - **GHL Assign Warm to Secondary Rep**  
    - Assigns contact to secondaryRepUserId  
  - **GHL Tag Cold Lead**  
    - Tags contact as "Cold Lead"  
  - **GHL Assign Cold to Nurture Team**  
    - Assigns contact to nurtureTeamUserId  
  - **Sticky Note4**  
    - Summarizes routing, tagging, assignment, and notification process  
  - **Edge Cases:**  
    - API failures (auth errors, network issues) during tagging or assignment  
    - Missing or invalid user IDs in config causing assignment errors  
    - Slack API downtime or message send failures  
    - Score field missing or malformed causing routing logic to fail

---

#### 1.5 Webhook Response

- **Overview:**  
  Sends a JSON response back to the original webhook caller (GHL) indicating success, with lead ID, score, and category.

- **Nodes Involved:**  
  - Webhook Response

- **Node Details:**  
  - **Webhook Response**  
    - Type: Respond to Webhook  
    - Responds with JSON object: success flag, message, contactId, score, category  
    - Inputs: From Slack notification or direct assignment branches (end of workflow)  
    - Role: Confirms processing completion to GHL  
    - Edge Cases: Response failures if triggered prematurely or with missing data

---

### 3. Summary Table

| Node Name                 | Node Type                 | Functional Role                          | Input Node(s)                  | Output Node(s)                     | Sticky Note                            |
|---------------------------|---------------------------|----------------------------------------|-------------------------------|-----------------------------------|--------------------------------------|
| GHL New Contact Trigger    | Webhook                   | Entry point: receives new GHL contacts | None                          | Workflow Configuration            | ## Trigger & Config (Sticky Note1)   |
| Workflow Configuration     | Set                       | Loads API keys, user IDs, Slack channel| GHL New Contact Trigger        | GHL Get Contact Details           | ## Trigger & Config (Sticky Note1)   |
| GHL Get Contact Details    | HTTP Request              | Fetches full contact data from GHL     | Workflow Configuration         | GHL Fetch Engagement Data         | ## Contact & Engagement Data (Sticky Note2) |
| GHL Fetch Engagement Data  | HTTP Request              | Fetches engagement activity data       | GHL Get Contact Details        | Clean Contact Data                | ## Contact & Engagement Data (Sticky Note2) |
| Clean Contact Data         | Set                       | Cleans and normalizes contact and engagement data | GHL Fetch Engagement Data       | Prepare AI Input                  | ## Contact & Engagement Data (Sticky Note2) |
| Prepare AI Input           | Set                       | Prepares structured JSON for AI scoring| Clean Contact Data             | AI Lead Scoring (GPT-4)           | ## Contact & Engagement Data (Sticky Note2) |
| AI Lead Scoring (GPT-4)    | OpenAI (Langchain)        | Uses GPT-4 to score leads and categorize| Prepare AI Input               | Parse Score to Numeric            | ## AI Lead Scoring (Sticky Note3)    |
| Parse Score to Numeric     | Code (JavaScript)         | Parses AI response, normalizes score   | AI Lead Scoring (GPT-4)        | IF Score >= 80 (Hot)              | ## AI Lead Scoring (Sticky Note3)    |
| IF Score >= 80 (Hot)       | If                        | Checks if lead score ≥ 80 (Hot)        | Parse Score to Numeric         | GHL Tag Hot Lead, IF Score >= 40 (Warm) | ## Route & Assign (Sticky Note4)     |
| GHL Tag Hot Lead           | HTTP Request              | Tags contact as Hot Lead in GHL         | IF Score >= 80 (Hot)           | GHL Assign Hot to Top Rep         | ## Route & Assign (Sticky Note4)     |
| GHL Assign Hot to Top Rep  | HTTP Request              | Assigns Hot leads to top sales rep      | GHL Tag Hot Lead               | Slack Notify Hot Lead             | ## Route & Assign (Sticky Note4)     |
| Slack Notify Hot Lead      | Slack                     | Sends Slack alert for Hot leads         | GHL Assign Hot to Top Rep      | Webhook Response                 | ## Route & Assign (Sticky Note4)     |
| IF Score >= 40 (Warm)      | If                        | Checks if score ≥ 40 (Warm)              | IF Score >= 80 (Hot)           | GHL Tag Warm Lead, GHL Tag Cold Lead | ## Route & Assign (Sticky Note4)     |
| GHL Tag Warm Lead          | HTTP Request              | Tags contact as Warm Lead                 | IF Score >= 40 (Warm)          | GHL Assign Warm to Secondary Rep  | ## Route & Assign (Sticky Note4)     |
| GHL Assign Warm to Secondary Rep | HTTP Request        | Assigns Warm leads to secondary rep      | GHL Tag Warm Lead              | Webhook Response                 | ## Route & Assign (Sticky Note4)     |
| GHL Tag Cold Lead          | HTTP Request              | Tags contact as Cold Lead                 | IF Score >= 40 (Warm)          | GHL Assign Cold to Nurture Team   | ## Route & Assign (Sticky Note4)     |
| GHL Assign Cold to Nurture Team | HTTP Request          | Assigns Cold leads to nurture team       | GHL Tag Cold Lead              | Webhook Response                 | ## Route & Assign (Sticky Note4)     |
| Webhook Response           | Respond to Webhook        | Sends final success response back to GHL| Slack Notify Hot Lead, GHL Assign Warm to Secondary Rep, GHL Assign Cold to Nurture Team | None            |                                      |
| Sticky Note                | Sticky Note               | Documentation on overall workflow        | None                          | None                            | See respective content in block analysis |
| Sticky Note1               | Sticky Note               | Documentation on trigger & config        | None                          | None                            | See block 1.1                        |
| Sticky Note2               | Sticky Note               | Documentation on data retrieval & prep   | None                          | None                            | See block 1.2                        |
| Sticky Note3               | Sticky Note               | Documentation on AI scoring               | None                          | None                            | See block 1.3                        |
| Sticky Note4               | Sticky Note               | Documentation on routing & assignment     | None                          | None                            | See block 1.4                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `ghl-new-contact`  
   - Purpose: Receive new contact data from GoHighLevel webhook

2. **Create Set Node for Workflow Configuration**  
   - Type: Set  
   - Parameters to add as strings:  
     - `ghlApiBaseUrl`: e.g., `https://rest.gohighlevel.com/v1`  
     - `ghlApiKey`: Your GoHighLevel API key  
     - `topRepUserId`: User ID for top sales rep (Hot leads)  
     - `secondaryRepUserId`: User ID for secondary sales rep (Warm leads)  
     - `nurtureTeamUserId`: User ID for nurture team (Cold leads)  
     - `slackChannel`: Slack channel ID (e.g., `#sales`) for Hot lead alerts  
   - Connect Webhook Trigger output to this node

3. **Add HTTP Request Node to Get Contact Details**  
   - Method: GET  
   - URL: `{{$json["ghlApiBaseUrl"]}}/contacts/{{$json["body"]["contact_id"]}}` (use expression referencing Workflow Configuration and webhook)  
   - Headers: `Authorization: Bearer {{$json.ghlApiKey}}`  
   - Connect Workflow Configuration node output here

4. **Add HTTP Request Node to Fetch Engagement Data**  
   - Method: GET  
   - URL: `{{$json["ghlApiBaseUrl"]}}/contacts/{{$json["body"]["contact_id"]}}/activities`  
   - Headers: same as above  
   - Connect output of Get Contact Details node here

5. **Add Set Node to Clean Contact Data**  
   - Extract and assign fields with fallback defaults, e.g.:  
     - `contactId`: from contact details JSON, fallback empty string  
     - `name`, `email`, `phone`, `source`  
     - `tags` as array  
     - `customFields` as object  
     - `engagementData` from engagement fetch node  
   - Connect output of Fetch Engagement Data node here

6. **Add Set Node to Prepare AI Input**  
   - Create an object named `leadData` with sub-objects `demographics` and `behavior` containing relevant fields from cleaned data  
   - Connect Clean Contact Data output here

7. **Add OpenAI (Langchain) Node for AI Lead Scoring**  
   - Model: GPT-4 (`gpt-4o`)  
   - Instructions prompt as per scoring criteria (buying intent, engagement, behavior, quality, urgency)  
   - Input: JSON stringified `leadData` from previous node  
   - Credentials: configure OpenAI API key  
   - Connect output of Prepare AI Input node here

8. **Add Code Node to Parse AI Score**  
   - JavaScript code to parse AI JSON response safely, fallback to regex extraction if needed  
   - Normalize score between 1-100  
   - Include contact metadata for routing  
   - Connect AI Lead Scoring output here

9. **Add If Node for Hot Lead Check (Score ≥ 80)**  
   - Condition: score exists and score ≥ 80  
   - Connect output of Parse Score node here

10. **Add If Node for Warm Lead Check (Score ≥ 40)**  
    - Condition: score exists and score ≥ 40  
    - Connect False branch of Hot Lead If node here

11. **Add HTTP Request Nodes to Tag Leads in GHL**  
    - Hot Lead Tag: PUT `/contacts/{contactId}` with JSON body `{"tags":["Hot Lead"]}`  
    - Warm Lead Tag: PUT `/contacts/{contactId}` with JSON body `{"tags":["Warm Lead"]}`  
    - Cold Lead Tag: PUT `/contacts/{contactId}` with JSON body `{"tags":["Cold Lead"]}`  
    - Headers: Authorization Bearer + Content-Type application/json  
    - Connect Hot If True branch to Hot Tag node  
    - Connect Warm If True branch to Warm Tag node  
    - Connect Warm If False branch to Cold Tag node

12. **Add HTTP Request Nodes to Assign Leads to Users**  
    - Hot Lead Assign: PUT `/contacts/{contactId}/assign` with JSON body `{"userId": topRepUserId}`  
    - Warm Lead Assign: PUT `/contacts/{contactId}/assign` with JSON body `{"userId": secondaryRepUserId}`  
    - Cold Lead Assign: PUT `/contacts/{contactId}/assign` with JSON body `{"userId": nurtureTeamUserId}`  
    - Connect each Tag node output to respective Assign node

13. **Add Slack Node to Notify Hot Leads**  
    - Use Slack channel ID from configuration  
    - Message includes name, email, score, source, confidence, reasoning  
    - Credentials: OAuth2 Slack account  
    - Connect Hot Lead Assign output to Slack node

14. **Add Respond to Webhook Node**  
    - Respond with JSON: success, message, contactId, score, category  
    - Connect Slack node output and Assign nodes for Warm and Cold leads here to ensure all paths finalize with a response

15. **Add Sticky Notes as documentation nodes** (optional but recommended)  
    - Document each logical block’s purpose and usage for maintenance

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| This workflow provides a scalable system to leverage AI for lead scoring integrated tightly with GoHighLevel CRM and Slack notifications. | Workflow description |
| Setup requires creating a webhook in GoHighLevel pointing to this workflow’s URL and configuring API keys and user IDs correctly. | Setup instructions in Sticky Note |
| AI prompt can be customized to adjust scoring criteria or thresholds. | AI Lead Scoring node configuration |
| Slack alerts ensure rapid sales team follow-up on high-priority leads. | Slack Notify Hot Lead node |
| For GoHighLevel API details, refer to https://rest.gohighlevel.com/ | External API documentation |
| OpenAI GPT-4 usage requires API key and managing token usage to avoid rate limits. | OpenAI API considerations |
| Slack integration requires OAuth2 credentials with chat:write scope. | Slack API documentation |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. All processing complies with applicable content policies and does not contain illegal, offensive, or protected material. All data handled is legal and public.