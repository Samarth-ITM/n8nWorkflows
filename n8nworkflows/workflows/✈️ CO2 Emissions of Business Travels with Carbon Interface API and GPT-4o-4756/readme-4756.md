✈️ CO2 Emissions of Business Travels with Carbon Interface API and GPT-4o

https://n8nworkflows.xyz/workflows/---co2-emissions-of-business-travels-with-carbon-interface-api-and-gpt-4o-4756


# ✈️ CO2 Emissions of Business Travels with Carbon Interface API and GPT-4o

### 1. Workflow Overview

This n8n workflow automates the processing and carbon emissions calculation of business travel flights based on incoming Gmail emails. It targets business users who want to track and assess the environmental impact of their travel itineraries by extracting structured trip details from flight confirmation emails, recording them in Google Sheets, and enriching the data with CO2 emissions estimates using the Carbon Interface API.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception (Gmail Trigger):** Watches a Gmail inbox for new emails related to travel bookings.
- **1.2 AI Parsing of Trip Details (AI Agent Parser & Structured Output Parser):** Uses an AI language model (GPT-4o-mini) to extract structured trip information including flight segments, traveler name, trip purpose, and booking details.
- **1.3 Flight Data Splitting and Recording (Split Out, Loop Over Flights, Record Flights Information):** Splits the extracted flights into individual records and logs them into a Google Sheet.
- **1.4 CO2 Emissions Estimation (Collect CO2 Emissions HTTP Request, Load Results):** For each flight, queries the Carbon Interface API to estimate carbon emissions and appends/updates the results in the Google Sheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers the entire workflow upon receiving a new email in a configured Gmail inbox. It enables automatic processing of business travel emails.

- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Trigger node for Gmail  
    - Configuration: Polls Gmail every minute for new emails; downloads no attachments; uses OAuth2 credentials (to be configured)  
    - Input: None (trigger)  
    - Output: Emits new email data as JSON including email text body  
    - Version: 1.2  
    - Edge Cases:  
      - Authentication errors if OAuth2 token expires or is invalid.  
      - Rate limit or Gmail API throttling issues.  
      - Emails without expected structured content may cause downstream parsing failures.

---

#### 2.2 AI Parsing of Trip Details

- **Overview:**  
  This block uses an AI agent powered by GPT-4o-mini to parse the free-form email text and extract a detailed structured JSON object describing the trip, including traveler name, trip purpose, flight segments with airport codes, dates, and booking statuses.

- **Nodes Involved:**  
  - AI Agent Parser  
  - Structured Output Parser  
  - OpenAI Chat Model2 (used as language model for AI Agent Parser)

- **Node Details:**

  - **OpenAI Chat Model2**  
    - Type: Language model node using OpenAI GPT-4o-mini variant  
    - Configuration: Model set to `gpt-4o-mini`; requires OpenAI API key credentials  
    - Input: Receives prompt and system instructions from AI Agent Parser  
    - Output: Language model response JSON  
    - Version: 1.2  
    - Edge Cases:  
      - API key issues, quota limits, or network timeouts.  
      - Model response may be incomplete or malformed if prompt is ambiguous.

  - **AI Agent Parser**  
    - Type: LangChain AI Agent node  
    - Configuration:  
      - System message instructs extraction of a strict JSON object with specified fields from the email text.  
      - The prompt uses the email body text from Gmail Trigger.  
      - Includes output parser linked to Structured Output Parser node.  
    - Input: Email JSON from Gmail Trigger  
    - Output: Parsed structured JSON object with trip details (flights array, traveler info, booking flags)  
    - Version: 1.9  
    - Edge Cases:  
      - Failure to extract valid JSON due to unexpected email format.  
      - Missing or malformed airport codes.  
      - Partial data if email content is incomplete.

  - **Structured Output Parser**  
    - Type: LangChain output parser node  
    - Configuration: JSON schema example provided to validate and structure AI output.  
    - Input: Raw AI Agent output  
    - Output: Validated and structured JSON object usable downstream  
    - Version: 1.2  
    - Edge Cases:  
      - Parsing errors if AI output deviates from expected JSON format.

---

#### 2.3 Flight Data Splitting and Recording

- **Overview:**  
  The workflow splits the list of flights extracted from the AI output into individual flight records, loops over them in batches, and appends each flight’s detailed info into a Google Sheet for record-keeping.

- **Nodes Involved:**  
  - Split Out  
  - Loop Over Flights  
  - Record Flights Information

- **Node Details:**

  - **Split Out**  
    - Type: Data manipulation node to split array fields into separate items  
    - Configuration: Splits `output.flights` array into individual flight objects  
    - Input: Parsed trip details JSON from AI Agent Parser  
    - Output: Individual flight JSON objects  
    - Version: 1  
    - Edge Cases:  
      - Empty or missing flights array causing no output.

  - **Loop Over Flights**  
    - Type: SplitInBatches node for batch processing  
    - Configuration: Processes flights one by one (default batch size 1)  
    - Input: Individual flight records from Split Out  
    - Output: Sequential single flight items for downstream processing  
    - Version: 3  
    - Edge Cases:  
      - Large number of flights may increase execution time or hit limits.

  - **Record Flights Information**  
    - Type: Google Sheets node  
    - Configuration:  
      - Appends flight data to a specified Google Sheet (document ID provided).  
      - Maps fields such as traveler name, trip ID, flight dates, airport codes, booking flags, and flight number.  
      - Requires Google Sheets OAuth2 credentials configured.  
      - Sheet name identified by gid=0 (likely default sheet).  
    - Input: Single flight JSON from Loop Over Flights, plus reference to AI Agent Parser output for additional trip fields  
    - Output: Confirmation of data append operation  
    - Version: 4.6  
    - Edge Cases:  
      - API quota or permission errors.  
      - Mismatched schema or missing fields.  
      - Sheet not found or access denied.

---

#### 2.4 CO2 Emissions Estimation and Result Loading

- **Overview:**  
  For each recorded flight, this block calls the Carbon Interface API to estimate CO2 emissions based on flight route details, then updates the Google Sheet with the emissions, distance, and estimation timestamp.

- **Nodes Involved:**  
  - Collect CO2 Emissions  
  - Load Results

- **Node Details:**

  - **Collect CO2 Emissions**  
    - Type: HTTP Request node  
    - Configuration:  
      - POST request to `https://www.carboninterface.com/api/v1/estimates`  
      - Sends JSON body with flight type, passengers = 1, and legs specifying departure and destination airport codes from Google Sheets data.  
      - Headers include `Authorization: Bearer YOUR_API_KEY` (must be replaced with actual API key) and `Content-Type: application/json`.  
    - Input: Flight record JSON from Record Flights Information node  
    - Output: API JSON response with carbon emissions data (carbon_kg, distance_value, estimated_at)  
    - Version: 4.2  
    - Edge Cases:  
      - Invalid or missing API key causing 401 Unauthorized.  
      - API rate limits or service downtime.  
      - Invalid airport codes or malformed request causing 400 errors.

  - **Load Results**  
    - Type: Google Sheets node  
    - Configuration:  
      - Appends or updates rows in the same Google Sheet, matching on Trip ID.  
      - Maps CO2 emissions, distance, and estimation time fields from Carbon Interface API response.  
      - Requires Google Sheets OAuth2 credentials.  
      - Sheet identified by gid=0.  
    - Input: API response JSON from Collect CO2 Emissions, linked with flight record data.  
    - Output: Confirmation of append/update operation.  
    - Version: 4.6  
    - Edge Cases:  
      - Sheet update conflicts or permission issues.  
      - API inconsistencies causing data mismatches.

---

### 3. Summary Table

| Node Name               | Node Type                         | Functional Role                              | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                                      |
|-------------------------|----------------------------------|----------------------------------------------|-----------------------|--------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger           | n8n-nodes-base.gmailTrigger      | Workflow trigger on new email reception      | None                  | AI Agent Parser          | Workflow is triggered by new Gmail emails related to flight schedules. Setup requires Gmail API credentials.                    |
| AI Agent Parser         | @n8n/n8n-nodes-langchain.agent  | Extract structured trip details from email  | Gmail Trigger         | Split Out                | Uses GPT-4o-mini to parse detailed JSON from email with strict formatting instructions.                                         |
| Structured Output Parser| @n8n/n8n-nodes-langchain.outputParserStructured | Validate and parse AI output JSON         | AI Agent Parser (ai_outputParser) | AI Agent Parser | Validates AI output against JSON schema to ensure structured trip data.                                                         |
| OpenAI Chat Model2      | @n8n/n8n-nodes-langchain.lmChatOpenAi | Language model node providing AI processing | AI Agent Parser (ai_languageModel) | AI Agent Parser | Requires OpenAI API credentials. Uses GPT-4o-mini model.                                                                          |
| Split Out               | n8n-nodes-base.splitOut          | Split flights array into individual flights  | AI Agent Parser       | Loop Over Flights        |                                                                                                                               |
| Loop Over Flights       | n8n-nodes-base.splitInBatches    | Process flights one at a time                 | Split Out             | Record Flights Information, Load Results (via second output) |                                                                                                                               |
| Record Flights Information | n8n-nodes-base.googleSheets     | Append flight info to Google Sheet            | Loop Over Flights     | Collect CO2 Emissions    | Requires Google Sheets API credentials. Maps comprehensive flight and trip details.                                             |
| Collect CO2 Emissions   | n8n-nodes-base.httpRequest       | Call Carbon Interface API for CO2 estimation | Record Flights Information | Load Results          | Requires Carbon Interface API key. Sends flight leg info, receives emissions data.                                              |
| Load Results            | n8n-nodes-base.googleSheets      | Append or update Google Sheet with emissions | Collect CO2 Emissions | Loop Over Flights (2nd output) | Updates sheet with CO2 kg, distance, and estimation time. Requires Google Sheets API credentials.                               |
| Sticky Note1            | n8n-nodes-base.stickyNote        | Documentation note on Gmail Trigger setup    | None                  | None                     | Workflow is triggered by new Gmail emails related to flight schedules. Setup requires Gmail API credentials.                    |
| Sticky Note2            | n8n-nodes-base.stickyNote        | Documentation note on AI Agent setup          | None                  | None                     | Explains AI agent parsing setup including OpenAI model and prompt instructions.                                                 |
| Sticky Note             | n8n-nodes-base.stickyNote        | Documentation on Google Sheets and Carbon Interface API usage | None | None | Explains how to configure Google Sheets nodes and Carbon Interface API for distance and CO2 data retrieval. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node:**
   - Node Type: Gmail Trigger  
   - Poll for new emails every minute  
   - OAuth2 credentials: Configure with Gmail API OAuth2 credentials for your mailbox  
   - Do not download attachments  
   - Purpose: Trigger workflow on new travel-related emails  

2. **Add OpenAI Chat Model Node:**
   - Node Type: LangChain OpenAI Chat Model  
   - Model: `gpt-4o-mini`  
   - Credentials: Configure with valid OpenAI API key  
   - Purpose: Provide the language model for AI parsing  

3. **Add AI Agent Parser Node:**
   - Node Type: LangChain Agent  
   - Text Input: Use expression `{{ $json.text }}` from Gmail Trigger (email body)  
   - System Prompt: Configure to instruct extraction of a strict JSON object with fields: traveler_name, trip_purpose, trip_id, flights array (departure and return), hotel_booked, ground_transport_booked  
   - Output Parser: Link to Structured Output Parser node  
   - Connect OpenAI Chat Model node as language model  
   - Purpose: Extract structured trip JSON from email  

4. **Add Structured Output Parser Node:**
   - Node Type: LangChain Output Parser Structured  
   - JSON Schema Example: Provide a sample JSON structure matching expected output  
   - Connect as output parser to AI Agent Parser node  
   - Purpose: Validate and structure AI-generated JSON  

5. **Add Split Out Node:**
   - Node Type: Split Out  
   - Field to Split Out: `output.flights` (from AI Agent Parser output)  
   - Purpose: Split flight array into individual flight records  

6. **Add Loop Over Flights Node:**
   - Node Type: SplitInBatches  
   - Default batch size (1)  
   - Connect output of Split Out node  
   - Purpose: Process each flight individually downstream  

7. **Add Google Sheets Node for Flight Recording:**
   - Node Type: Google Sheets (Append)  
   - Credentials: Configure with Google Sheets OAuth2 credentials  
   - Document ID: Set to your target Google Sheet file ID  
   - Sheet Name: Use sheet with gid=0 or your chosen sheet  
   - Map columns: Traveler Name, Trip Purpose, Trip ID (concatenate trip_id and flight type), Hotel Booked, Ground Transport Booked, Flight Type, From, From Code, To, To Code, Flight Date, Flight Number  
   - Input: Single flight JSON from Loop Over Flights plus trip-level data from AI Agent Parser  
   - Purpose: Record detailed flight info for each segment  

8. **Add HTTP Request Node for CO2 Emissions:**
   - Node Type: HTTP Request (POST)  
   - URL: `https://www.carboninterface.com/api/v1/estimates`  
   - Headers:  
     - `Authorization: Bearer YOUR_API_KEY` (replace with your Carbon Interface API key)  
     - `Content-Type: application/json`  
   - JSON Body:  
     ```json
     {
       "type": "flight",
       "passengers": 1,
       "legs": [
         {
           "departure_airport": "{{ $json['From Code'] }}",
           "destination_airport": "{{ $json['To Code'] }}"
         }
       ]
     }
     ```  
   - Input: Flight record from Google Sheets Append node  
   - Purpose: Fetch CO2 emissions and distance data for each flight  

9. **Add Google Sheets Node to Load Results:**
   - Node Type: Google Sheets (Append or Update)  
   - Credentials: Same Google Sheets OAuth2 credentials  
   - Document ID and Sheet Name: Same as flight recording node  
   - Matching Columns: Trip ID (to update existing rows)  
   - Map fields: CO2 (kg), Distance (km), Estimation Time from API response  
   - Input: HTTP Request node output (CO2 emissions data)  
   - Purpose: Append or update emissions data in the sheet  

10. **Connect nodes accordingly:**
    - Gmail Trigger → AI Agent Parser  
    - AI Agent Parser → Structured Output Parser (as output parser)  
    - AI Agent Parser → Split Out  
    - Split Out → Loop Over Flights  
    - Loop Over Flights → Record Flights Information  
    - Record Flights Information → Collect CO2 Emissions (HTTP Request)  
    - Collect CO2 Emissions → Load Results  
    - Loop Over Flights also connects second output to Load Results to finish batch processing  

11. **Set up credentials:**
    - Gmail OAuth2 for Gmail Trigger  
    - OpenAI API key for Chat Model node  
    - Google Sheets OAuth2 for both Google Sheets nodes  
    - Carbon Interface API key in HTTP Request node header  

12. **Test the workflow:**
    - Send a sample business trip confirmation email to the configured Gmail inbox with flight details  
    - Verify AI parsing outputs correct structured JSON  
    - Confirm flight data is appended in Google Sheet  
    - Confirm CO2 emissions data is fetched and updated in Google Sheet  

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Gmail Trigger Node documentation for setup and OAuth2 configuration                                                         | https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.gmailtrigger                               |
| AI Agent Node with Chat Model instructions including prompt design and output parsing                                        | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent                       |
| Carbon Interface API key signup and API documentation                                                                        | https://docs.carboninterface.com/#/                                                                             |
| Google Sheets Node configuration and OAuth2 setup instructions                                                              | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets                                   |
| Workflow designed to process flight schedules from emails, extract trip details, and calculate environmental impact metrics | Internal project for automating business travel carbon footprint tracking                                         |

---

This reference document fully describes the structure, node configuration, data flow, and integration points for the ✈️ CO2 Emissions of Business Travels workflow, enabling full understanding, reproduction, and customization.