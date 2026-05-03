Real-Time Lead Qualification, Scoring & CRM Integration System using Jotform

https://n8nworkflows.xyz/workflows/real-time-lead-qualification--scoring---crm-integration-system-using-jotform-9815


# Real-Time Lead Qualification, Scoring & CRM Integration System using Jotform

### 1. Workflow Overview

This n8n workflow implements a **Real-Time Lead Qualification, Scoring & CRM Integration System** driven by lead data captured via Jotform. Its primary function is to convert raw lead submissions into qualified sales opportunities using AI, route leads intelligently to the appropriate sales representatives, integrate enriched data into a CRM, and notify both sales reps and leads instantly. The workflow is designed to optimize lead conversion speed and accuracy, enabling a 5-minute response time and significantly increasing sales conversion rates.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Data Extraction:** Captures form submissions from Jotform and extracts structured lead data fields.
- **1.2 AI Lead Scoring (BANT Framework):** Uses an AI agent to analyze the lead against Budget, Authority, Need, and Timeline criteria, producing a detailed qualification score and insights.
- **1.3 Intelligent Lead Routing:** Applies custom JavaScript logic to assign leads to sales reps based on score, territory, industry expertise, and workload.
- **1.4 CRM Integration & Contact Creation:** Sends enriched lead data to a CRM (HubSpot in this case) to create a new contact record.
- **1.5 Notifications:** Sends instant, detailed email notifications to the assigned sales rep and confirmation emails to the lead.
- **1.6 Analytics & Tracking:** Logs all lead and scoring data to Google Sheets for performance monitoring and ROI analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Data Extraction

- **Overview:**  
  This block listens for incoming lead submissions from a Jotform form and extracts relevant lead information into structured JSON fields for downstream processing.

- **Nodes Involved:**  
  - Jotform Trigger  
  - Extract Lead Data (Set node)

- **Node Details:**

  - **Jotform Trigger**  
    - *Type:* Trigger node for Jotform form submissions  
    - *Configuration:* Listens on form ID `252862984356471` via webhook  
    - *Key Expressions:* N/A  
    - *Inputs:* External HTTP webhook (Jotform)  
    - *Outputs:* Raw JSON of form submission  
    - *Potential Failures:* Webhook misconfiguration, API auth issues, form ID changes  
    - *Credentials:* JotForm API OAuth2

  - **Extract Lead Data**  
    - *Type:* Set node to map and rename incoming form fields  
    - *Configuration:* Extracts specific fields from `$json.rawRequest` such as companyName, contactName, email, phone, companySize, budgetRange, timeline, industry, currentSolution, painPoints, plus submissionId and timestamp  
    - *Key Expressions:* `={{ $json.rawRequest['q3_companyName'] }}`, `$now.toISO()`  
    - *Inputs:* From Jotform Trigger  
    - *Outputs:* Structured lead JSON with normalized keys  
    - *Potential Failures:* Missing fields in submission, expression evaluation errors

#### 2.2 AI Lead Scoring (BANT Framework)

- **Overview:**  
  This block uses an AI agent (via LangChain) to apply the BANT qualification framework (Budget, Authority, Need, Timeline) to the extracted lead data, producing a comprehensive JSON qualification including scoring, tier assignment, insights, and recommended actions.

- **Nodes Involved:**  
  - OpenAI Chat Model (gpt-4.1-mini)  
  - AI Lead Scoring (BANT) (LangChain Agent)  
  - Parse Lead Score (Set node)

- **Node Details:**

  - **OpenAI Chat Model**  
    - *Type:* Language model node using OpenAI GPT-4.1-mini  
    - *Configuration:* Model selected: `gpt-4.1-mini`  
    - *Credentials:* OpenAI API key  
    - *Inputs:* None directly; connected as AI language model for agent node  
    - *Outputs:* Model responses for agent prompt  
    - *Potential Failures:* API rate limits, credential expiration, model unavailability

  - **AI Lead Scoring (BANT)**  
    - *Type:* LangChain agent node for conversational AI processing  
    - *Configuration:* Uses a system message positioning the agent as a B2B sales qualification expert; prompt includes all extracted lead fields and requests JSON output with detailed BANT scores, tier, and recommendations  
    - *Key Expressions:* Template strings referencing extracted lead data (e.g., `{{ $json.companyName }}`)  
    - *Inputs:* Lead data from Extract Lead Data node, AI model from OpenAI Chat Model  
    - *Outputs:* JSON-formatted lead qualification including scores, tier, insights, red flags, and next steps  
    - *Potential Failures:* AI response format errors, timeout, malformed JSON output

  - **Parse Lead Score**  
    - *Type:* Set node (empty in config, presumably for further processing or mapping)  
    - *Configuration:* Uses expression mode, likely to parse or normalize AI output for routing  
    - *Inputs:* From AI Lead Scoring node  
    - *Outputs:* Prepared lead score and qualification data for routing  
    - *Potential Failures:* Expression or data format errors

#### 2.3 Intelligent Lead Routing

- **Overview:**  
  This block implements custom JavaScript logic to assign leads to sales representatives based on lead score, industry fit, territory, deal size capacity, and rep workload, adhering to predefined routing rules.

- **Nodes Involved:**  
  - Intelligent Routing Logic (Code node)  

- **Node Details:**

  - **Intelligent Routing Logic**  
    - *Type:* Code node executing custom JavaScript  
    - *Configuration:*  
      - Defines sales team members categorized by rep level (senior, midLevel, SDR) with attributes: name, email, territories, industries, minimum deal size, and current lead load  
      - Assigns rep level per lead score: ≥75 = senior, 50-74 = midLevel, else SDR  
      - Filters reps by industry match or 'All' industry coverage  
      - Selects rep with lowest current lead count to balance workload  
      - Outputs assigned rep details, assignment reason, timestamp, and full lead data  
    - *Key Expressions:* References to previous node data like `$('Extract Lead Data').first().json.industry`  
    - *Inputs:* From Parse Lead Score  
    - *Outputs:* Lead data enriched with assigned rep info and routing metadata  
    - *Potential Failures:* Missing data fields, JavaScript runtime errors, no eligible reps matching

#### 2.4 CRM Integration & Contact Creation

- **Overview:**  
  This block creates a new contact record in a CRM system (HubSpot) using the enriched lead and routing data to ensure the lead is officially tracked and actionable in sales pipelines.

- **Nodes Involved:**  
  - Create CRM Contact (HTTP Request node)

- **Node Details:**

  - **Create CRM Contact**  
    - *Type:* HTTP Request node  
    - *Configuration:*  
      - POST to HubSpot contacts API endpoint `https://api.hubapi.com/crm/v3/objects/contacts`  
      - Sends JSON body with complete lead info, AI scores, BANT details, assigned rep, next steps, and timestamps  
      - Uses predefined HubSpot API OAuth2 credentials  
    - *Inputs:* From Intelligent Routing Logic  
    - *Outputs:* API response from HubSpot (success or error)  
    - *Potential Failures:* API authentication errors, rate limits, malformed JSON, network issues  
    - *Credentials:* HubSpot API OAuth2

#### 2.5 Notifications

- **Overview:**  
  This block sends email notifications to both the assigned sales rep and the lead, ensuring timely communication and engagement.

- **Nodes Involved:**  
  - Notify Sales Rep (Gmail node)  
  - Send Lead Confirmation (Gmail node)

- **Node Details:**

  - **Notify Sales Rep**  
    - *Type:* Gmail node for sending email  
    - *Configuration:*  
      - Sends to assigned rep email (`{{ $json.assignedRepEmail }}`)  
      - Email subject includes lead tier and score  
      - HTML email body with full lead profile, BANT breakdown, insights, red flags, next steps, and talking points  
      - Uses Gmail OAuth2 credentials  
    - *Inputs:* From Intelligent Routing Logic  
    - *Outputs:* Email send response  
    - *Potential Failures:* Email sending errors, invalid email addresses, OAuth token expiry  
    - *Credentials:* Gmail OAuth2

  - **Send Lead Confirmation**  
    - *Type:* Gmail node for sending email  
    - *Configuration:*  
      - Sends confirmation email to lead’s email address (`{{ $json.email }}`)  
      - Friendly thank you message, assigned rep contact info, and next steps timeline  
      - Uses same Gmail OAuth2 credentials as above  
    - *Inputs:* From Intelligent Routing Logic  
    - *Outputs:* Email send response  
    - *Potential Failures:* Same as Notify Sales Rep node

#### 2.6 Analytics & Tracking

- **Overview:**  
  This block logs all lead details, AI scoring, assignment, and timestamps into a Google Sheets spreadsheet for ongoing analytics, rep performance monitoring, and ROI tracking.

- **Nodes Involved:**  
  - Log to Tracking Sheet (Google Sheets node)

- **Node Details:**

  - **Log to Tracking Sheet**  
    - *Type:* Google Sheets node  
    - *Configuration:*  
      - Appends or updates a row in the sheet named `Lead_Tracking` within the specified Google Sheet document (ID placeholder `YOUR_GOOGLE_SHEET_ID`)  
      - Maps multiple lead and scoring fields including email, phone, status, urgency, industry, rep info, scores, timestamps, pain points, and conversion metrics  
      - Uses OAuth2 credentials for Google Sheets API  
    - *Inputs:* From Intelligent Routing Logic  
    - *Outputs:* Response from Google Sheets API on append/update  
    - *Potential Failures:* API auth issues, sheet ID or name misconfiguration, quota limits  
    - *Credentials:* Google Sheets OAuth2

---

### 3. Summary Table

| Node Name               | Node Type                            | Functional Role                              | Input Node(s)             | Output Node(s)                                 | Sticky Note                                                                                              |
|-------------------------|------------------------------------|----------------------------------------------|---------------------------|-----------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Sticky Note             | Sticky Note                        | Lead Qualification & Distribution overview  | -                         | -                                             | 🎯 Lead Qualification & Distribution: Purpose, features, ROI overview                                  |
| Sticky Note1            | Sticky Note                        | Jotform Lead Capture Fields info             | -                         | -                                             | 📝 Jotform Lead Capture Fields and link to create form                                                |
| Jotform Trigger         | Trigger (Jotform)                  | Captures lead submissions from Jotform      | -                         | Extract Lead Data                              |                                                                                                        |
| Extract Lead Data       | Set                               | Extract and normalize lead data fields       | Jotform Trigger           | AI Lead Scoring (BANT)                         |                                                                                                        |
| Sticky Note2            | Sticky Note                        | AI Lead Scoring (BANT) explanation            | -                         | -                                             | 🤖 AI Lead Scoring (BANT): Analysis and output details                                                |
| OpenAI Chat Model       | LangChain LM (OpenAI)              | Provides GPT-4 model for AI Lead Scoring     | -                         | AI Lead Scoring (BANT) (AI model input)       |                                                                                                        |
| AI Lead Scoring (BANT)  | LangChain Agent                   | AI BANT analysis and scoring                  | Extract Lead Data, OpenAI Chat Model | Parse Lead Score                              |                                                                                                        |
| Parse Lead Score        | Set                               | Prepare AI scoring output for routing        | AI Lead Scoring (BANT)    | Intelligent Routing Logic                      |                                                                                                        |
| Sticky Note3            | Sticky Note                        | Intelligent Lead Routing explanation          | -                         | -                                             | 🎯 Intelligent Lead Routing: Routing logic and factors considered                                     |
| Intelligent Routing Logic | Code                             | Assign leads to sales reps based on criteria | Parse Lead Score          | Create CRM Contact, Notify Sales Rep, Send Lead Confirmation, Log to Tracking Sheet |                                                                                                        |
| Sticky Note4            | Sticky Note                        | CRM Integration overview                      | -                         | -                                             | 📊 CRM Integration: Contact creation details and CRM compatibility                                   |
| Create CRM Contact      | HTTP Request (HubSpot API)         | Creates contact in CRM                         | Intelligent Routing Logic | -                                             |                                                                                                        |
| Sticky Note5            | Sticky Note                        | Instant Rep Notification explanation          | -                         | -                                             | 📧 Instant Rep Notification: Email content and response time                                        |
| Notify Sales Rep        | Gmail                             | Sends email notification to assigned sales rep | Intelligent Routing Logic | -                                             |                                                                                                        |
| Sticky Note6            | Sticky Note                        | Lead Confirmation Email explanation            | -                         | -                                             | 🔔 Lead Confirmation Email: Content and trust-building                                              |
| Send Lead Confirmation  | Gmail                             | Sends confirmation email to lead              | Intelligent Routing Logic | -                                             |                                                                                                        |
| Sticky Note7            | Sticky Note                        | Analytics & Tracking overview                  | -                         | -                                             | 📊 Analytics & Tracking: Logging to Google Sheets and metrics tracking                              |
| Log to Tracking Sheet   | Google Sheets                     | Logs lead and routing data for analysis       | Intelligent Routing Logic | -                                             |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Jotform Trigger node:**  
   - Select node type: JotForm Trigger  
   - Configure with your Jotform API credentials (OAuth2)  
   - Set form ID to your Jotform lead capture form ID (e.g., `252862984356471`)  
   - Activate webhook for new submissions  

2. **Add Extract Lead Data node (Set):**  
   - Connect from Jotform Trigger output  
   - Map all relevant form fields from `$json.rawRequest` to new keys:  
     - companyName ← `q3_companyName`  
     - contactName ← `q4_contactName`  
     - email ← `q5_email`  
     - phone ← `q6_phone`  
     - companySize ← `q7_companySize`  
     - budgetRange ← `q8_budgetRange`  
     - timeline ← `q9_timeline`  
     - industry ← `q10_industry`  
     - currentSolution ← `q11_currentSolution`  
     - painPoints ← `q12_painPoints`  
     - submissionId ← `submissionID`  
     - submittedAt ← current timestamp (`{{$now.toISO()}}`)  

3. **Set up OpenAI Chat Model node:**  
   - Add LangChain LM node using OpenAI GPT-4.1-mini model  
   - Add OpenAI API credentials (API key)  

4. **Add AI Lead Scoring (BANT) node (LangChain Agent):**  
   - Connect AI model input from OpenAI Chat Model  
   - Connect lead data input from Extract Lead Data  
   - Configure prompt with BANT framework instructions and include all lead fields dynamically (`{{ $json.fieldName }}`)  
   - Use a system message defining the agent as a B2B sales qualification expert specialized in BANT  
   - Expect JSON output with scores, tier, insights, and recommended actions  

5. **Add Parse Lead Score node (Set):**  
   - Connect from AI Lead Scoring output  
   - Configure to parse or prepare AI JSON output for routing (if needed, set expressions to extract key properties)  

6. **Add Intelligent Routing Logic node (Code):**  
   - Connect from Parse Lead Score  
   - Paste JavaScript code defining sales teams, rep levels, filtering logic, and selection by workload  
   - Use expressions to access lead data fields (industry, leadScore, companySize)  
   - Output assigned rep details, assignment reason, and enriched lead data  

7. **Add Create CRM Contact node (HTTP Request):**  
   - Connect from Intelligent Routing Logic output  
   - Configure POST request to CRM API endpoint (e.g., HubSpot: `https://api.hubapi.com/crm/v3/objects/contacts`)  
   - Set authentication using your CRM’s OAuth2 credentials  
   - Compose JSON body including all lead data, AI scoring, BANT details, assigned rep, next steps, and timestamps  

8. **Add Notify Sales Rep node (Gmail):**  
   - Connect from Intelligent Routing Logic output  
   - Configure Gmail node with OAuth2 credentials  
   - Set recipient to assigned rep’s email (`{{ $json.assignedRepEmail }}`)  
   - Compose rich HTML email with lead profile, BANT breakdown, insights, next steps, and talking points  
   - Set appropriate subject line referencing lead tier and score  

9. **Add Send Lead Confirmation node (Gmail):**  
   - Connect from Intelligent Routing Logic output  
   - Configure Gmail node with same credentials  
   - Set recipient to lead’s email (`{{ $json.email }}`)  
   - Compose friendly thank you HTML email with assigned rep contact info and next steps  
   - Set subject line as "Thank You for Your Interest!"  

10. **Add Log to Tracking Sheet node (Google Sheets):**  
    - Connect from Intelligent Routing Logic output  
    - Configure Google Sheets node with OAuth2 credentials  
    - Set operation to append or update in the sheet named `Lead_Tracking` inside your Google Sheet document (provide document ID)  
    - Map all relevant fields for tracking: email, phone, status, urgency, industry, rep info, scores, timestamps, pain points, conversion data  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                   | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Workflow transforms lead chaos into systematic qualification and routing, providing a 5-minute response time and up to 300% conversion increase.              | Sticky Note (initial overview)                                                                              |
| Jotform fields correspond to common lead capture inputs; form creation is free at Jotform with a partner link provided.                                       | Sticky Note1                                                                                               |
| The AI agent uses BANT methodology, outputting a structured JSON with detailed scoring and recommended sales actions.                                         | Sticky Note2                                                                                               |
| Intelligent routing matches leads to sales reps by score, territory, industry expertise, and workload balancing.                                              | Sticky Note3                                                                                               |
| CRM integration supports HubSpot and can be extended to Salesforce, Pipedrive, or any API-compatible CRM.                                                     | Sticky Note4                                                                                               |
| Email notifications to sales reps include detailed lead profiles and AI insights; lead confirmation emails build trust and set expectations.                 | Sticky Notes5, 6                                                                                           |
| Google Sheets logging enables analytics, rep performance tracking, and lead source ROI measurement.                                                           | Sticky Note7                                                                                               |
| Jotform trigger requires correct webhook setup and API credentials; ensure form ID matches your lead capture form.                                            | Jotform Trigger node context                                                                                |
| OpenAI and Gmail nodes require OAuth2 credentials with appropriate scopes; watch for token expiry or API limits.                                              | Credentials notes                                                                                            |
| HubSpot API integration assumes valid OAuth2 token and correct API permissions for contact creation.                                                          | Create CRM Contact node context                                                                              |
| Google Sheets integration requires sharing the spreadsheet and OAuth2 credentials with edit rights; document ID and sheet name must be accurate.              | Log to Tracking Sheet node context                                                                           |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.