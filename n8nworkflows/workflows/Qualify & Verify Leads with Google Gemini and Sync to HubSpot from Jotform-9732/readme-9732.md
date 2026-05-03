Qualify & Verify Leads with Google Gemini and Sync to HubSpot from Jotform

https://n8nworkflows.xyz/workflows/qualify---verify-leads-with-google-gemini-and-sync-to-hubspot-from-jotform-9732


# Qualify & Verify Leads with Google Gemini and Sync to HubSpot from Jotform

### 1. Workflow Overview

This workflow automates the process of qualifying and verifying leads submitted via a Jotform, enriching those leads with AI-powered data extraction and summarization, and synchronizing the enriched leads to HubSpot CRM. It also sends an internal notification email with key lead details and AI-generated insights to the sales team for immediate follow-up.

**Target Use Cases:**  
- Marketing and sales teams who collect lead information via Jotform and want to automate lead qualification and enrichment.  
- Organizations aiming to reduce manual data entry errors, verify lead email validity, and prioritize follow-up with AI-generated summaries.  
- Teams using HubSpot CRM that require automated contact creation and update based on enriched lead data.

**Logical Blocks:**  
- **1.1 Input Reception:** Capture lead data from Jotform form submissions.  
- **1.2 Lead Email Verification:** Verify lead email addresses using Verifalia to ensure deliverability.  
- **1.3 Website Data Extraction:** Crawl the submitted company website and extract structured business information using AI.  
- **1.4 AI Lead Summarization:** Use Google Gemini AI to generate a concise summary of the lead’s potential.  
- **1.5 HubSpot Synchronization:** Create or update the contact record in HubSpot CRM with enriched data.  
- **1.6 Internal Notification:** Send an email notification to the sales team with lead details and AI summary.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives new lead data when a prospect submits the Jotform. This triggers the entire workflow.

**Nodes Involved:**  
- JotForm Trigger

**Node Details:**  
- **JotForm Trigger**  
  - Type: Trigger node for Jotform submissions  
  - Configuration: Listens to form with ID `252856264643060`  
  - Inputs: Webhook from Jotform on form submission  
  - Outputs: JSON data containing lead fields such as firstname, lastname, email, phone, website, and note  
  - Edge Cases:  
    - Missing or malformed form data  
    - Delay or failure in webhook trigger delivery  
  - Version: 1

---

#### 1.2 Lead Email Verification

**Overview:**  
Validates the lead’s email address using the Verifalia API to ensure it is deliverable and not fake or temporary.

**Nodes Involved:**  
- Lead Verification

**Node Details:**  
- **Lead Verification (HTTP Request)**  
  - Type: HTTP Request node  
  - Configuration:  
    - POST request to `https://api.verifalia.com/v2.7/email-validations`  
    - Payload includes the lead’s email from the Jotform trigger  
    - Headers include `Accept: application/json` and `authorization` (API Key header)  
  - Inputs: JSON from JotForm Trigger with email field  
  - Outputs: API response indicating email validity  
  - Edge Cases:  
    - API authentication failure (wrong or missing API key)  
    - Network timeouts or API rate limiting  
    - Invalid or missing email address in input  
  - Version: 4.2

---

#### 1.3 Website Data Extraction

**Overview:**  
Fetches the lead’s company website HTML and extracts key structured information such as company name, country, industry, and a brief summary using an AI-powered information extractor.

**Nodes Involved:**  
- HTTP Request  
- Information Extractor  
- Google Gemini Chat Model (used as the underlying AI language model for extraction)

**Node Details:**  
- **HTTP Request**  
  - Type: HTTP Request node  
  - Configuration:  
    - GET request to the website URL supplied by the lead in the Jotform submission  
    - Allows unauthorized certificates to accommodate less-secure sites  
  - Inputs: Lead’s website URL from JotForm Trigger  
  - Outputs: Website HTML content  
  - Edge Cases:  
    - Invalid or unreachable website URL  
    - Website blocking automated requests or requiring authentication  
    - SSL certificate errors (allowed by configuration)  
  - Version: 4.2  

- **Information Extractor**  
  - Type: LangChain Information Extractor node  
  - Configuration:  
    - Input text is the website HTML content from the HTTP Request node  
    - Requests extraction of attributes:  
      - `company_name` (required): Company name or website name  
      - `country`: Company’s country  
      - `industry` (required): Industry sector (e.g., tech, finance)  
      - `summary` (required): Short summary of website services  
    - Uses Google Gemini Chat Model as the underlying language model  
  - Inputs: Website HTML text  
  - Outputs: JSON with extracted attributes  
  - Edge Cases:  
    - AI model failing to extract meaningful data  
    - Missing or ambiguous website content  
  - Version: 1.2  

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini Chat Model node  
  - Configuration: Default parameters, used implicitly by Information Extractor  
  - Inputs: Internal calls from Information Extractor  
  - Outputs: AI model completions for extraction  
  - Edge Cases:  
    - API quota exceeded or authentication failure  
    - Latency or transient errors from AI service  
  - Version: 1

---

#### 1.4 AI Lead Summarization

**Overview:**  
Creates a concise, personalized summary of the lead’s potential based on all collected and enriched data, acting as a virtual sales assistant.

**Nodes Involved:**  
- Create or update a contact  
- Message a model

**Node Details:**  
- **Create or update a contact (HubSpot)**  
  - Type: HubSpot node (contact management)  
  - Configuration:  
    - Uses lead’s email as key identifier  
    - Fields populated from both Jotform and Information Extractor: firstname, lastname, email, phone, website, note, country, industry, company name  
  - Inputs: JSON with extracted and original lead data  
  - Outputs: HubSpot contact creation or update confirmation  
  - Edge Cases:  
    - Authentication failure with HubSpot API (OAuth2 or API key)  
    - Data validation errors (e.g., invalid email format)  
    - Rate limits or API downtime  
  - Version: 2.1  

- **Message a model (Google Gemini)**  
  - Type: LangChain Google Gemini node  
  - Configuration:  
    - Model: `models/gemini-2.5-flash`  
    - Input message: a writing assistant prompt that includes all lead details and extracted attributes, requesting a short summary of lead prospectiveness  
    - Output in JSON format  
  - Inputs: Lead data including AI-extracted fields and HubSpot confirmation output  
  - Outputs: AI-generated summary text  
  - Edge Cases:  
    - Model request failures or invalid formatting  
    - API quota or authentication issues  
  - Version: 1

---

#### 1.5 Internal Notification

**Overview:**  
Sends an email to the internal sales team containing the lead details and the AI-generated summary for immediate, informed follow-up.

**Nodes Involved:**  
- Send a message (Gmail)

**Node Details:**  
- **Send a message (Gmail)**  
  - Type: Gmail node  
  - Configuration:  
    - Sends a plain text email  
    - Subject: "New Lead Captured"  
    - Message body: includes AI summary and all lead data fields (name, email, phone, country, industry, company name, website, note) dynamically populated from previous nodes  
  - Inputs: AI summary from Message a model and lead data from JotForm Trigger and Information Extractor  
  - Outputs: Email sending confirmation  
  - Edge Cases:  
    - OAuth2 authentication failure or token expiration  
    - Email sending limits exceeded  
    - Missing email recipients (configured in credentials or defaults)  
  - Version: 2.1

---

### 3. Summary Table

| Node Name              | Node Type                             | Functional Role                          | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                  |
|------------------------|-------------------------------------|----------------------------------------|------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| JotForm Trigger        | jotFormTrigger                      | Captures lead form submissions         | —                      | Lead Verification        | Typeform trigger:  Gets trigger after capturing lead data                                                    |
| Lead Verification      | httpRequest                        | Verifies lead email address validity   | JotForm Trigger        | HTTP Request             | Verify Leads on Email if they are not fake or temporary emails                                               |
| HTTP Request           | httpRequest                        | Fetches lead’s website HTML             | Lead Verification      | Information Extractor    | Extract the website html and get country, industry, website summary and company name                         |
| Information Extractor  | informationExtractor (LangChain)   | Extracts structured data from website  | HTTP Request           | Create or update a contact | Extract the website html and get country, industry, website summary and company name                         |
| Google Gemini Chat Model| lmChatGoogleGemini (LangChain)     | Underlying AI model for data extraction| — (used internally)    | Information Extractor    |                                                                                                              |
| Create or update a contact | hubspot                        | Creates or updates HubSpot contact     | Information Extractor   | Message a model           | Create or Update contact in hubspot                                                                          |
| Message a model        | googleGemini (LangChain)            | Generates AI lead summary               | Create or update a contact | Send a message          | AI will summarize these data about the lead                                                                  |
| Send a message         | gmail                             | Sends notification email to sales team | Message a model        | —                        | Send your self a notification                                                                                  |
| Sticky Note            | stickyNote                        | Comments/annotations                    | —                      | —                        | Typeform trigger:  Gets trigger after capturing lead data                                                    |
| Sticky Note1           | stickyNote                        | Comments/annotations                    | —                      | —                        | Create or Update contact in hubspot                                                                           |
| Sticky Note2           | stickyNote                        | Comments/annotations                    | —                      | —                        | Send your self a notification                                                                                  |
| Sticky Note3           | stickyNote                        | Comments/annotations                    | —                      | —                        | AI will summarize these data about the lead                                                                   |
| Sticky Note4           | stickyNote                        | Comments/annotations                    | —                      | —                        | Verify Leads on Email if they are not fake or temporary emails                                               |
| Sticky Note5           | stickyNote                        | Comments/annotations                    | —                      | —                        | Extract the website html and get country, industry, website summary and company name                         |
| Sticky Note6           | stickyNote                        | Workflow overview and description      | —                      | —                        | ## AI-Enhanced Lead Qualification & HubSpot Synchronization (Jotform to HubSpot)... [full description above] |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger node**  
   - Type: JotForm Trigger  
   - Configure with your Jotform API Key credentials  
   - Select Form ID: `252856264643060` (replace with your form’s ID)  
   - This node triggers when the form is submitted.

2. **Add HTTP Request node for Lead Verification**  
   - Connect input from JotForm Trigger  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.verifalia.com/v2.7/email-validations`  
   - Headers:  
     - `Accept: application/json`  
     - `authorization: Bearer <Your Verifalia API Key>`  
   - Body (JSON):  
     ```json
     {
       "entries": [
         {
           "inputData": {{$json.email}}
         }
       ]
     }
     ```  
   - Enable "Send Body" and "Send Headers" options.  
   - Set credentials with your Verifalia API Key.

3. **Add HTTP Request node to fetch lead’s website HTML**  
   - Connect input from Lead Verification node  
   - Type: HTTP Request  
   - Method: GET (default)  
   - URL: `={{$('JotForm Trigger').item.json.website}}`  
   - Options: Allow unauthorized certificates (to bypass SSL issues)  

4. **Add Information Extractor node (LangChain)**  
   - Connect input from HTTP Request (website fetch)  
   - Type: Information Extractor  
   - Text input: `={{ $json.data }}` (the HTML content from previous node)  
   - Attributes to extract:  
     - `company_name` (required)  
     - `country`  
     - `industry` (required)  
     - `summary` (required)  
   - Configure to use Google Gemini Chat Model as the language model.

5. **Add Google Gemini Chat Model node**  
   - This node is called internally by Information Extractor for AI processing  
   - No manual configuration needed unless customizing model parameters.

6. **Add HubSpot node to create or update contact**  
   - Connect input from Information Extractor  
   - Type: HubSpot (Contact Management)  
   - Authentication: Configure OAuth2 or API Key for HubSpot  
   - Map fields:  
     - Email: `={{ $('JotForm Trigger').item.json.email }}`  
     - First Name: `={{ $('JotForm Trigger').item.json.firstname }}`  
     - Last Name: `={{ $('JotForm Trigger').item.json.lastname }}`  
     - Phone Number: `={{ $('JotForm Trigger').item.json.phone }}`  
     - Website URL: `={{ $('JotForm Trigger').item.json.website }}`  
     - Company Name: `={{ $json.company_name }}`  
     - Country: `={{ $json.country }}`  
     - Industry: `={{ $json.industry }}`  
     - Message/Note: `={{ $('JotForm Trigger').item.json.note }}`

7. **Add Google Gemini node to generate lead summary**  
   - Connect input from HubSpot node  
   - Type: Google Gemini (LangChain)  
   - Model ID: `models/gemini-2.5-flash`  
   - Message content (template):  
     ```
     You are a writing assistant for email, which reviews the lead data and writes a short summary to notify me if the lead is prospective.

     Here are the details:

     Name: {{ $('JotForm Trigger').item.json.firstname }} {{ $('JotForm Trigger').item.json.lastname }}
     Email: {{ $('JotForm Trigger').item.json.email }}
     Phone: {{ $('JotForm Trigger').item.json.phone }}
     Country: {{ $('Information Extractor').item.json.country ?? "" }}
     Industry: {{ $('Information Extractor').item.json.industry }}
     Company Name: {{ $('Information Extractor').item.json.company_name }}
     Summary: {{ $('Information Extractor').item.json.summary }}

     Website: {{ $('JotForm Trigger').item.json.website }}
     Note: {{ $('JotForm Trigger').item.json.note }}
     ```  
   - Set output as JSON.

8. **Add Gmail node to send internal notification email**  
   - Connect input from Google Gemini (Message a model) node  
   - Type: Gmail  
   - Authentication: Configure OAuth2 for Gmail  
   - Subject: `New Lead Captured`  
   - Message (plain text):  
     ```
     {{$json.content.parts[0].text}}

     Here are the details:

     Name: {{ $('JotForm Trigger').item.json.firstname }} {{ $('JotForm Trigger').item.json.lastname }}
     Email: {{ $('JotForm Trigger').item.json.email }}
     Phone: {{ $('JotForm Trigger').item.json.phone }}
     Country: {{ $('Information Extractor').item.json.country ?? "" }}
     Industry: {{ $('Information Extractor').item.json.industry }}
     Company Name: {{ $('Information Extractor').item.json.company_name }}
     Summary: {{ $('Information Extractor').item.json.summary }}

     Website: {{ $('JotForm Trigger').item.json.website }}
     Note: {{ $('JotForm Trigger').item.json.note }}
     ```

9. **Connect nodes in order:**  
   JotForm Trigger → Lead Verification → HTTP Request (website fetch) → Information Extractor → Create or update a contact (HubSpot) → Message a model (Google Gemini) → Send a message (Gmail)

10. **Add Sticky Notes (optional):**  
    - Add descriptive sticky notes near relevant nodes as per original workflow for documentation and clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                                                                              |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses multiple APIs and requires valid credentials configured in n8n: Jotform API key, Verifalia API key, Google Gemini/PaLM API key (for AI nodes), HubSpot OAuth2/API key, and Gmail OAuth2 credentials. Make sure to keep these credentials secure and refreshed as needed.                                                                                                                                                                                                                                                                                                                                                                   | Credential setup in n8n                                                                                                                                                       |
| The workflow filters out leads with invalid or temporary emails by leveraging Verifalia’s email validation API, reducing CRM pollution and increasing sales efficiency.                                                                                                                                                                                                                                                                                                                                                                                                                                                                             | Lead quality assurance                                                                                          |
| The AI Information Extractor node uses Google Gemini as the underlying language model to parse unstructured website HTML and extract structured business data. This depends on the quality and accessibility of the website content.                                                                                                                                                                                                                                                                                                                                                                              | AI-powered data enrichment                                                                                      |
| The AI summarization step produces a short summary for sales teams, enabling more personalized and prioritized outreach. Adjust the prompt as needed to suit your business context.                                                                                                                                                                                                                                                                                                                                                                                                                                                                | AI prompt engineering                                                                                           |
| HubSpot node requires appropriate permissions to create or update contacts; ensure the connected HubSpot account has these rights.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    | CRM integration                                                                                                 |
| Gmail node sends notifications internally; ensure OAuth2 consent is granted and that sending limits are respected to avoid disruptions.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Internal notification                                                                                           |
| Sticky notes in the workflow provide helpful guidance and should be maintained or expanded to assist future maintainers or automation agents.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | Workflow documentation                                                                                          |
| For more on n8n's integrations and AI nodes, visit the official documentation: https://docs.n8n.io/                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | n8n documentation                                                                                               |

---

**Disclaimer:** The text above is generated exclusively from an automated n8n workflow and adheres strictly to content policies. It contains no illegal or offensive material. All data processed is legal and publicly accessible.