🚚 CO2 Emissions of Freight Shipments with Carbon Interface API and GPT-4o

https://n8nworkflows.xyz/workflows/---co2-emissions-of-freight-shipments-with-carbon-interface-api-and-gpt-4o-4757


# 🚚 CO2 Emissions of Freight Shipments with Carbon Interface API and GPT-4o

### 1. Workflow Overview

This workflow automates the extraction, processing, and recording of CO₂ emissions data for freight shipments by integrating email parsing, AI-driven data extraction, carbon footprint estimation, and data logging. It is designed for logistics teams or environmental analysts who want to monitor shipment emissions based on incoming shipment confirmation emails.

The workflow is composed of four main logical blocks:

- **1.1 Input Reception:** Triggered by new incoming shipment emails in Gmail.
- **1.2 AI Processing:** Uses an AI Agent with GPT-4o-mini to extract structured shipment details from email content.
- **1.3 Emission Estimation:** Sends shipment details to the Carbon Interface API to estimate freight CO₂ emissions.
- **1.4 Data Storage:** Records the shipment details and emissions data into a Google Sheet for tracking and reporting.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Listens for new emails in a designated Gmail account to initiate the workflow whenever a shipment confirmation is received.

- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**  
  - **Gmail Trigger**  
    - Type: Trigger node, listens for new Gmail messages.  
    - Configuration: Polls every minute, no filters on emails, attachments not downloaded.  
    - Inputs: None (trigger node).  
    - Outputs: Passes email data (including email body text) downstream.  
    - Edge cases: Gmail API authentication errors, quota limits, network timeouts.  
    - Notes: Requires Gmail OAuth2 credentials configured.  
    - Sticky Note: Guidance on Gmail Trigger setup and credential requirements.  

#### 2.2 AI Processing

- **Overview:**  
  Processes the raw email text using an AI Agent powered by GPT-4o-mini to extract structured shipment information in JSON format, enabling downstream usage for emission calculations and record keeping.

- **Nodes Involved:**  
  - AI Agent Parser  
  - OpenAI Chat Model2  
  - Structured Output Parser

- **Node Details:**  
  - **AI Agent Parser**  
    - Type: LangChain AI Agent node.  
    - Configuration: Uses a system prompt customized for logistics shipment extraction specifying exact JSON fields to extract (shipment number, pickup/delivery details, cargo weight, driving distance, etc.). Only returns the JSON object without explanation.  
    - Inputs: Email text from Gmail Trigger.  
    - Outputs: Structured JSON with shipment details.  
    - Key expressions: Uses `{{ $json.text }}` from email, defines strict JSON output format, normalizes dates to ISO 8601, removes units from numbers.  
    - Edge cases: AI output inconsistencies, malformed emails, API rate limits or failures, token limits.  
    - Sub-workflow: None.  

  - **OpenAI Chat Model2**  
    - Type: LangChain OpenAI Chat Model node.  
    - Configuration: Model set to "gpt-4o-mini" for parsing tasks.  
    - Inputs: Feeds AI Agent Parser as the language model backend.  
    - Outputs: Provides AI-generated responses to the Agent.  
    - Credentials: Requires OpenAI API key.  
    - Edge cases: Authentication errors, request timeouts, throttling.  

  - **Structured Output Parser**  
    - Type: LangChain structured output parser node.  
    - Configuration: Uses a JSON schema example matching the expected shipment data structure to validate and parse AI Agent output.  
    - Inputs: Connects as output parser to AI Agent Parser.  
    - Outputs: Validated structured data for use in subsequent nodes.  
    - Edge cases: Parsing failures if AI output deviates from schema, JSON syntax errors.  

#### 2.3 Emission Estimation

- **Overview:**  
  Takes structured shipment data and requests CO₂ emission estimates from the Carbon Interface API based on cargo weight (converted to grams), shipment distance, and transport method.

- **Nodes Involved:**  
  - Record Shipment Information  
  - Collect CO2 Emissions

- **Node Details:**  
  - **Record Shipment Information**  
    - Type: Google Sheets node (append or update operation).  
    - Configuration: Maps structured shipment fields (shipment number, pickup and destination info, cargo details, times, etc.) into a Google Sheet identified by document ID and sheet name. Uses shipment_number as the matching column for updates.  
    - Inputs: Structured shipment JSON from AI Agent Parser.  
    - Outputs: Passes data downstream for emission calculation.  
    - Credentials: Requires Google Sheets OAuth2 API credentials.  
    - Edge cases: Google API authentication failures, rate limits, sheet permission issues, data mapping mismatches.  

  - **Collect CO2 Emissions**  
    - Type: HTTP Request node.  
    - Configuration: POST request to Carbon Interface API endpoint `/api/v1/estimates`.  
      - Payload includes:  
        - type: "shipping"  
        - weight_value: converted cargo weight tons to grams (tons × 1,000,000)  
        - weight_unit: "g"  
        - distance_value: driving_distance_km  
        - distance_unit: "km"  
        - transport_method: "truck"  
      - Headers: Authorization with Bearer token (user to replace `YOUR_API_KEY`), Content-Type `application/json`.  
    - Inputs: Shipment data from Record Shipment Information.  
    - Outputs: JSON response containing estimated CO₂ emissions data.  
    - Edge cases: API key invalid or missing, network errors, rate limiting, malformed requests, unexpected API response.  
    - Sticky Note: Instructions on obtaining and setting Carbon Interface API key.  

#### 2.4 Data Storage

- **Overview:**  
  Logs the returned CO₂ emissions data from the Carbon Interface API back into the same Google Sheet, appending or updating the shipment record with emission values and estimation timestamps.

- **Nodes Involved:**  
  - Load Results

- **Node Details:**  
  - **Load Results**  
    - Type: Google Sheets node (append or update operation).  
    - Configuration: Maps the API response fields `carbon_kg` (CO2 emissions), `estimated_at` (timestamp) alongside shipment_number into the same Google Sheet. Uses shipment_number as the key for matching rows.  
    - Inputs: CO₂ data from Collect CO2 Emissions node.  
    - Outputs: None (terminal node).  
    - Credentials: Google Sheets OAuth2 API credentials required.  
    - Edge cases: Same as Record Shipment Information node (API errors, permission issues).  
    - Sticky Note: Instructions on configuring Google Sheets for data logging.  

---

### 3. Summary Table

| Node Name                | Node Type                              | Functional Role                                   | Input Node(s)        | Output Node(s)               | Sticky Note                                                                                                                     |
|--------------------------|--------------------------------------|-------------------------------------------------|----------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger            | n8n-nodes-base.gmailTrigger           | Trigger workflow on new shipment confirmation emails | None                 | AI Agent Parser             | Setup guidance for Gmail Trigger and credentials [Learn more](https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.gmailtrigger) |
| AI Agent Parser          | @n8n/n8n-nodes-langchain.agent       | Extract structured shipment data from email text | Gmail Trigger        | Record Shipment Information  | Setup guidance for AI Agent using Chat Model with custom prompt [Learn more](https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent)|
| OpenAI Chat Model2       | @n8n/n8n-nodes-langchain.lmChatOpenAi| Provides GPT-4o-mini model backend for AI Agent  | AI Agent Parser (ai_languageModel) | AI Agent Parser (ai_languageModel) | Requires OpenAI API credentials                                                                                              |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Validates and structures AI output JSON          | AI Agent Parser (ai_outputParser) | AI Agent Parser (ai_outputParser) | None                                                                                                                        |
| Record Shipment Information | n8n-nodes-base.googleSheets         | Store shipment details into Google Sheet          | AI Agent Parser      | Collect CO2 Emissions        | Setup Google Sheets credentials, document ID, and sheet name; map shipment fields                                              |
| Collect CO2 Emissions    | n8n-nodes-base.httpRequest            | Call Carbon Interface API to estimate emissions   | Record Shipment Information | Load Results                 | Setup Carbon Interface API key in header (`Bearer YOUR_API_KEY`); POST JSON payload with shipment weight and distance          |
| Load Results             | n8n-nodes-base.googleSheets           | Append CO₂ emissions data to Google Sheet          | Collect CO2 Emissions | None                        | Setup Google Sheets credentials; map CO₂ fields and timestamps                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**  
   - Type: Gmail Trigger  
   - Credentials: Configure Gmail OAuth2 credentials with proper scopes for reading emails.  
   - Settings: Poll every minute, no filters unless desired, do not download attachments.  
   - Connect output to AI Agent Parser node.

2. **Create OpenAI Chat Model Node:**  
   - Type: LangChain OpenAI Chat Model  
   - Model: Select "gpt-4o-mini".  
   - Credentials: Add OpenAI API key credentials.  
   - No inputs connected yet.

3. **Create AI Agent Parser Node:**  
   - Type: LangChain AI Agent  
   - Parameters:  
     - Text input: Use expression to extract email body text: `{{ $json.text }}` from Gmail Trigger.  
     - System message: Provide a detailed prompt instructing the AI to extract shipment details strictly in JSON format with specified fields (shipment number, pickup/delivery info, weight, times, etc.). Normalize dates to ISO 8601 and remove units from numbers.  
   - Connect OpenAI Chat Model as the language model backend (`ai_languageModel` input).  
   - Add a structured output parser node and connect it as `ai_outputParser` to validate AI output.

4. **Create Structured Output Parser Node:**  
   - Type: LangChain Structured Output Parser  
   - Configuration: Provide JSON schema example matching the expected shipment data structure (as given in the overview).  
   - Connect output back to AI Agent Parser node.

5. **Create Record Shipment Information Node:**  
   - Type: Google Sheets  
   - Credentials: Add Google Sheets OAuth2 credentials.  
   - Operation: Append or update.  
   - Document ID: Provide Google Sheet ID where shipment data should be recorded.  
   - Sheet Name: Select the relevant sheet (e.g., "CO2 emissions").  
   - Mapping: Map fields from the AI Agent Parser output (e.g., shipment_number, pickup_address, cargo_weight_tons, etc.) to sheet columns. Use shipment_number as matching column for updates.  
   - Connect AI Agent Parser output to this node.

6. **Create Collect CO2 Emissions Node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://www.carboninterface.com/api/v1/estimates`  
   - Headers:  
     - Authorization: `Bearer YOUR_API_KEY` (replace YOUR_API_KEY with real Carbon Interface API key)  
     - Content-Type: `application/json`  
   - Body (JSON):  
     ```json
     {
       "type": "shipping",
       "weight_value": {{ $json.output.cargo_weight_tons * 1000000 }},
       "weight_unit": "g",
       "distance_value": {{ $json.output.driving_distance_km }},
       "distance_unit": "km",
       "transport_method": "truck"
     }
     ```  
   - Connect Record Shipment Information output to this node.

7. **Create Load Results Node:**  
   - Type: Google Sheets  
   - Credentials: Google Sheets OAuth2 credentials (same or different as above).  
   - Operation: Append or update.  
   - Document ID and Sheet Name: Same sheet as Record Shipment Information node.  
   - Mapping: Map returned CO2 emissions fields (`carbon_kg` → `co2_kg`, `estimated_at` → `co2_estimation_time`) and `shipment_number` for matching.  
   - Connect Collect CO2 Emissions output to this node.

8. **Add Sticky Notes for Documentation:**  
   - Add notes explaining each block and setup instructions for Gmail Trigger, AI Agent, Carbon Interface API, and Google Sheets. Include links to official documentation.

9. **Test Workflow:**  
   - Send a test shipment confirmation email to the configured Gmail account.  
   - Verify structured extraction, emission estimation, and data logging.  
   - Handle errors such as API authentication failures, parsing errors, and update configurations as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                            | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Gmail Trigger setup requires OAuth2 credentials with Gmail API scopes for reading emails.                                                                                                | https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.gmailtrigger                   |
| AI Agent node uses LangChain integration with OpenAI GPT-4o-mini model, requires OpenAI API key and custom system prompt for structured JSON extraction from shipment emails.            | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent          |
| Carbon Interface API provides shipment CO₂ emission estimates based on weight and distance. Free API keys available with request limits. API key must be set in HTTP Request node header. | https://docs.carboninterface.com/#/                                                                |
| Google Sheets node requires OAuth2 credentials with appropriate access to the target Google Sheet and sheet permissions for appending/updating rows.                                     | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googleSheets                        |

---

**Disclaimer:**  
The provided content is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.