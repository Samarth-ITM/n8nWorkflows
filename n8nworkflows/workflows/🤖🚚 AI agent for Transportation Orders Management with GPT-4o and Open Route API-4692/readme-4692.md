🤖🚚 AI agent for Transportation Orders Management with GPT-4o and Open Route API

https://n8nworkflows.xyz/workflows/-----ai-agent-for-transportation-orders-management-with-gpt-4o-and-open-route-api-4692


# 🤖🚚 AI agent for Transportation Orders Management with GPT-4o and Open Route API

---

## 1. Workflow Overview

This workflow automates the management of transportation orders received by email for a logistics company. It processes shipment requests, extracts structured shipment data, enriches this data with geolocation and routing information, records it into Google Sheets, and sends a professional confirmation reply to the customer.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Triggered by incoming shipment request emails in Gmail.
- **1.2 AI Parsing of Shipment Data:** Uses an AI agent (GPT-4o-mini) to extract structured shipment details from email content.
- **1.3 Data Recording and Geocoding:** Records parsed shipment data into Google Sheets, queries Open Route API to obtain GPS coordinates for pickup and delivery addresses.
- **1.4 Route Calculation:** Calculates driving distance and estimated transit time between pickup and delivery points using the Open Route API.
- **1.5 Confirmation Email Generation:** Uses AI to compose a formal confirmation email and sends it as a reply to the original email.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block triggers the workflow when a new email arrives in a dedicated Gmail mailbox for shipment requests.

**Nodes Involved:**  
- Gmail Trigger

**Node Details:**

- **Gmail Trigger**  
  - Type: Trigger node (gmailTrigger)  
  - Role: Watches Gmail inbox for new emails every minute.  
  - Configuration: Polling mode set to every minute, no attachment downloads, no specific filters.  
  - Inputs: None (trigger)  
  - Outputs: Email data including subject and text body.  
  - Version: 1.2  
  - Edge Cases: Authentication failure with Gmail API, rate limits, or no new emails triggering the flow.

---

### 2.2 AI Parsing of Shipment Data

**Overview:**  
Extracts structured shipping information from the email content using an AI agent configured with GPT-4o-mini. The extracted data includes shipment number, pickup/delivery locations, timing, and sender info.

**Nodes Involved:**  
- AI Agent Parser  
- Structured Output Parser (attached to AI Agent Parser)  
- OpenAI Chat Model2 (language model node supporting AI Agent Parser)

**Node Details:**

- **AI Agent Parser**  
  - Type: LangChain AI agent node  
  - Role: Processes the email subject and body, extracting shipment details structured as JSON.  
  - Configuration:  
    - Input text combines email subject and body.  
    - System prompt defines the role as a logistics assistant extracting explicit shipment data only (no inference).  
    - Output format is a JSON object with defined fields such as shipment_number, pickup_location, expected_pickup_time, etc.  
  - Expressions: Uses `{{ $json.subject }}` and `{{ $json.text }}` from Gmail Trigger.  
  - Input: Email content from Gmail Trigger  
  - Output: JSON with structured shipment information.  
  - Version: 1.9  
  - Edge Cases: Missing fields set to `null`; possible parsing failure if email format is unexpected; API or quota errors with OpenAI.

- **Structured Output Parser**  
  - Type: LangChain output parser node  
  - Role: Validates and structures AI output as JSON according to a schema example.  
  - Input: Output from AI Agent Parser  
  - Output: Structured JSON used downstream.  
  - Version: 1.2  
  - Edge Cases: Parsing errors if AI output format deviates.

- **OpenAI Chat Model2**  
  - Type: LangChain chat model node using OpenAI GPT-4o-mini  
  - Role: Provides the language model capability for AI Agent Parser.  
  - Configuration: Model selected is `gpt-4o-mini`.  
  - Input: Prompt from AI Agent Parser  
  - Output: Text used by AI Agent Parser  
  - Version: 1.2  
  - Edge Cases: API key or quota issues, network timeouts.

---

### 2.3 Data Recording and Geocoding

**Overview:**  
Stores the parsed shipment data in a Google Sheet and enriches it by resolving pickup and delivery addresses to GPS coordinates via the Open Route API.

**Nodes Involved:**  
- Record Email Content (Google Sheets)  
- Collect Addresses (Google Sheets)  
- Query Open Route API Pickup (HTTP Request)  
- GPS Pickup (Set)  
- Save Pickup GPS (Google Sheets)  
- Query Open Route API Delivery (HTTP Request)  
- GPS Delivery (Set)  
- Save Delivery GPS (Google Sheets)  
- 5 sec (Wait)

**Node Details:**

- **Record Email Content**  
  - Type: Google Sheets node  
  - Role: Append or update shipment data into the "Orders" sheet, mapping fields extracted by AI (shipment number, addresses, timing, temperature control, destination, etc.)  
  - Configuration:  
    - Uses matching column `shipment_number` to update existing rows or append new ones.  
    - No type conversions enabled to preserve data integrity.  
  - Input: Structured output from AI Agent Parser  
  - Output: Confirmation of sheet update  
  - Version: 4.6  
  - Edge Cases: Google Sheets API auth failure, sheet or document not found, quota limits.

- **Collect Addresses**  
  - Type: Google Sheets node  
  - Role: Retrieve current shipment record by `shipment_number` to get addresses for geocoding.  
  - Configuration: Filter by shipment_number from AI output.  
  - Credentials: Google Sheets OAuth2  
  - Version: 4.6  
  - Edge Cases: No matching shipment found, auth failure.

- **Query Open Route API Pickup**  
  - Type: HTTP Request  
  - Role: Geocode pickup address to get GPS coordinates.  
  - Configuration:  
    - URL: Open Route API geocode endpoint  
    - Query parameters: API key, pickup address text, country restricted to France, OpenStreetMap source, size=1 (top result)  
    - Headers: Accept and Content-Type set for JSON  
  - Input: Pickup address from Collect Addresses  
  - Output: JSON geocode result  
  - Version: 4.2  
  - Edge Cases: API key invalid, no geocode result, rate limiting.

- **GPS Pickup**  
  - Type: Set node  
  - Role: Extracts longitude, latitude, borough, neighbourhood, and localadmin from geocode response for pickup location.  
  - Configuration: Assigns values from JSON path of response features array.  
  - Version: 3.4  
  - Edge Cases: Missing or malformed API response.

- **Save Pickup GPS**  
  - Type: Google Sheets node  
  - Role: Updates the shipment record with pickup GPS coordinates in the sheet.  
  - Configuration: Updates by matching shipment_number, fields: pickup_latitude, pickup_longitude.  
  - Version: 4.6  
  - Edge Cases: Sheet update failure, concurrency issues.

- **Query Open Route API Delivery**  
  - Same as Query Open Route API Pickup but for destination address.  
  - Edge cases same as pickup.

- **GPS Delivery**  
  - Same as GPS Pickup but for delivery address.

- **Save Delivery GPS**  
  - Same as Save Pickup GPS but saves destination_latitude and destination_longitude.

- **5 sec (Wait)**  
  - Type: Wait node  
  - Role: Pauses the flow for 5 seconds, likely to avoid race conditions or API rate limits before next steps.  
  - Version: 1.1

---

### 2.4 Route Calculation

**Overview:**  
Calculates driving distance and estimated transit time between pickup and delivery points for heavy goods vehicles, using the Open Route API.

**Nodes Involved:**  
- Collect Coordinates (Google Sheets)  
- Request Open Route API (HTTP Request)  
- Driving Time & Distance (Set)  
- Save Results (Google Sheets)

**Node Details:**

- **Collect Coordinates**  
  - Type: Google Sheets node  
  - Role: Retrieves shipment record by shipment_number to access GPS coordinates for routing.  
  - Configuration: Filter by shipment_number.  
  - Credentials: Google Sheets OAuth2  
  - Version: 4.6  
  - Edge Cases: No data found, auth errors.

- **Request Open Route API**  
  - Type: HTTP Request  
  - Role: Requests directions with driving mode "driving-hgv" (heavy goods vehicle) to get route info.  
  - Configuration:  
    - URL: Open Route API directions endpoint  
    - Query params: API key, start (pickup_longitude, pickup_latitude), end (destination_longitude, destination_latitude)  
    - Headers: Accept and Content-Type JSON  
  - Input: Coordinates from Collect Coordinates  
  - Output: JSON routing result with distance and duration  
  - Version: 4.2  
  - Edge Cases: Invalid coordinates, API key issues, rate limits.

- **Driving Time & Distance**  
  - Type: Set node  
  - Role: Parses driving distance (converted to kilometers with two decimals), duration (converted to minutes), and number of route steps from API response.  
  - Configuration: Uses JSONPath expressions on API response features for segments.  
  - Version: 3.4  
  - Edge Cases: Missing segments or unexpected response format.

- **Save Results**  
  - Type: Google Sheets node  
  - Role: Updates shipment record with driving_distance (km) and driving_time (minutes).  
  - Configuration: Matches by shipment_number.  
  - Credentials: Google Sheets OAuth2  
  - Version: 4.6  
  - Edge Cases: Sheet write errors.

---

### 2.5 Confirmation Email Generation

**Overview:**  
Generates a professional email confirmation with shipment details using an AI agent and sends a reply to the customer’s original email.

**Nodes Involved:**  
- Collect Shipment Information (Google Sheets)  
- AI Agent Reply  
- OpenAI Chat Model  
- Reply (Gmail)

**Node Details:**

- **Collect Shipment Information**  
  - Type: Google Sheets node  
  - Role: Retrieves the complete shipment record by shipment_number to supply data for confirmation email.  
  - Configuration: Filter by shipment_number.  
  - Credentials: Google Sheets OAuth2  
  - Version: 4.6  
  - Edge Cases: Missing record, auth failure.

- **AI Agent Reply**  
  - Type: LangChain AI agent node  
  - Role: Generates a clear, concise, and formal confirmation email body in HTML format with shipment details and contact info.  
  - Configuration:  
    - Input text template lists shipment details with placeholders from collected data.  
    - System message defines tone and content rules, emphasizing no invented data and English language output with HTML formatting.  
  - Version: 1.9  
  - Edge Cases: AI service availability, formatting errors.

- **OpenAI Chat Model**  
  - Type: LangChain chat model node with GPT-4o-mini  
  - Role: Provides the language model service to AI Agent Reply.  
  - Version: 1.2  
  - Edge Cases: API key issues, timeouts.

- **Reply (Gmail)**  
  - Type: Gmail node (gmail)  
  - Role: Sends the AI-generated HTML email as a reply to the original Gmail message.  
  - Configuration: Uses message ID from Gmail Trigger to reply, message body from AI Agent Reply.  
  - Version: 2.1  
  - Edge Cases: Gmail API auth failure, sending errors.

---

## 3. Summary Table

| Node Name                 | Node Type                      | Functional Role                                  | Input Node(s)           | Output Node(s)                 | Sticky Note                                                                                                        |
|---------------------------|--------------------------------|------------------------------------------------|-------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Gmail Trigger             | gmailTrigger                   | Trigger workflow on new shipment request email | -                       | AI Agent Parser               | ### 1. Workflow Trigger with Gmail Trigger Setup instructions and link                                              |
| AI Agent Parser           | langchain.agent               | Extract structured shipment info from email    | Gmail Trigger, OpenAI Chat Model2 | Collect Coordinates, Record Email Content, Collect Addresses, Collect Shipment Information | ### 2. AI Agent to parse shipment information from email with setup instructions and link                            |
| Structured Output Parser  | langchain.outputParserStructured | Validates AI output JSON structure              | AI Agent Parser          | AI Agent Parser               |                                                                                                                    |
| OpenAI Chat Model2        | langchain.lmChatOpenAi        | Provides AI language model for parsing          | AI Agent Parser          | AI Agent Parser               |                                                                                                                    |
| Record Email Content      | googleSheets                  | Append/update shipment data in Google Sheet     | AI Agent Parser          | -                             | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| Collect Addresses         | googleSheets                  | Retrieve shipment record by shipment_number     | AI Agent Parser          | Query Open Route API Pickup, Query Open Route API Delivery | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| Query Open Route API Pickup | httpRequest                  | Geocode pickup address to GPS                    | Collect Addresses        | GPS Pickup                    | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| GPS Pickup               | set                           | Extract coordinates and properties from geocode | Query Open Route API Pickup | Save Pickup GPS              | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| Save Pickup GPS          | googleSheets                  | Update Google Sheet with pickup GPS              | GPS Pickup               | 5 sec                        | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| 5 sec                    | wait                          | Pause flow 5 seconds                             | Save Pickup GPS          | -                             | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| Query Open Route API Delivery | httpRequest               | Geocode delivery address to GPS                   | Collect Addresses        | GPS Delivery                  | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| GPS Delivery             | set                           | Extract coordinates and properties from geocode | Query Open Route API Delivery | Save Delivery GPS           | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| Save Delivery GPS        | googleSheets                  | Update Google Sheet with delivery GPS             | GPS Delivery             | -                             | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| Collect Coordinates      | googleSheets                  | Retrieve shipment record coordinates              | AI Agent Parser          | Request Open Route API        | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| Request Open Route API   | httpRequest                   | Calculate driving distance and duration           | Collect Coordinates      | Driving Time & Distance       | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| Driving Time & Distance  | set                           | Extract driving distance and time from API output | Request Open Route API   | Save Results                  | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| Save Results             | googleSheets                  | Update sheet with driving distance and time       | Driving Time & Distance  | -                             | ### 3. Record Shipment Request Information and Open Route API usage instructions and links                         |
| Collect Shipment Information | googleSheets              | Retrieve full shipment record for email reply     | AI Agent Parser          | AI Agent Reply               | ### 4. Reply with confirmation setup instructions and link                                                        |
| AI Agent Reply           | langchain.agent               | Generate professional shipment confirmation email | Collect Shipment Information, OpenAI Chat Model | Reply                   | ### 4. Reply with confirmation setup instructions and link                                                        |
| OpenAI Chat Model        | langchain.lmChatOpenAi        | Provides AI language model for reply generation   | AI Agent Reply           | AI Agent Reply               | ### 4. Reply with confirmation setup instructions and link                                                        |
| Reply                    | gmail                        | Sends confirmation reply to customer               | AI Agent Reply, Gmail Trigger | -                          | ### 4. Reply with confirmation setup instructions and link                                                        |

---

## 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Configure to poll every minute for new emails.  
   - Set Gmail OAuth2 credentials.  
   - No filters or attachment downloads needed.

2. **Add OpenAI Chat Model Node for Parsing**  
   - Type: LangChain Chat Model (lmChatOpenAi)  
   - Model: gpt-4o-mini  
   - Provide OpenAI API credentials.

3. **Add AI Agent Parser Node**  
   - Type: LangChain Agent  
   - Connect input from Gmail Trigger.  
   - Use the OpenAI Chat Model node as language model.  
   - Set prompt to include email subject and body, ask to extract shipment data in JSON with explicit fields (shipment_number, pickup_location, etc.).  
   - Set system message defining role as logistics assistant extracting explicit data only.  
   - Enable output parser with a JSON schema example matching expected structure.

4. **Add Structured Output Parser Node**  
   - Type: LangChain Output Parser Structured  
   - Configure with JSON schema example matching the AI Agent Parser output.  
   - Connect output of AI Agent Parser to this node’s input parser.

5. **Add Google Sheets Node to Record Email Content**  
   - Operation: Append or Update  
   - Document: Select your Google Sheets file for "Transportation Orders"  
   - Sheet: Select your orders sheet (gid=0)  
   - Map fields from AI parsed output to columns: shipment_number, pickup_location, pickup_address, expected_pickup_time, temperature_control, destination_store_name, destination_address, expected_delivery_time.  
   - Use shipment_number as matching column for updates.  
   - Provide Google Sheets OAuth2 credentials.

6. **Add Google Sheets Node to Collect Addresses**  
   - Operation: Read rows with filter on shipment_number from AI output.  
   - Connect output of AI Agent Parser.  
   - Use same Google Sheets credentials and document.

7. **Add HTTP Request Node to Query Open Route API Pickup Address**  
   - URL: https://api.openrouteservice.org/geocode/search  
   - Query parameters: api_key (your Open Route API key), text (pickup_address), boundary.country=FR, sources=openstreetmap, size=1  
   - Headers: Content-Type and Accept as JSON  
   - Connect input from Collect Addresses node.

8. **Add Set Node to Extract Pickup GPS Coordinates**  
   - Extract longitude and latitude from API response JSON path features[0].geometry.coordinates array indexes 0 and 1.  
   - Optionally extract borough, neighbourhood, localadmin.  
   - Connect output of Pickup geocode node.

9. **Add Google Sheets Node to Save Pickup GPS**  
   - Operation: Update by shipment_number  
   - Map pickup_latitude and pickup_longitude from Set node output.  
   - Connect output of Set node.

10. **Add 5 Seconds Wait Node**  
    - Place after Save Pickup GPS node to avoid race conditions.

11. **Add HTTP Request Node to Query Open Route API Delivery Address**  
    - Same configuration as Pickup geocode node but text parameter uses destination_address from Collect Addresses.

12. **Add Set Node to Extract Delivery GPS Coordinates**  
    - Same as Pickup GPS extraction but for delivery address.

13. **Add Google Sheets Node to Save Delivery GPS**  
    - Update destination_latitude and destination_longitude in sheet.

14. **Add Google Sheets Node to Collect Coordinates**  
    - Read shipment record filtered by shipment_number for GPS coordinates.

15. **Add HTTP Request Node to Request Open Route API Directions**  
    - URL: https://api.openrouteservice.org/v2/directions/driving-hgv  
    - Query parameters: api_key, start (pickup_longitude, pickup_latitude), end (destination_longitude, destination_latitude)  
    - Headers: Content-Type and Accept JSON.

16. **Add Set Node to Extract Driving Distance and Duration**  
    - Extract distance (convert meters to km, 2 decimals) from features[0].properties.segments[0].distance  
    - Extract duration (seconds to minutes, 2 decimals) from features[0].properties.segments[0].duration  
    - Extract number of steps from segments[0].steps length.

17. **Add Google Sheets Node to Save Results**  
    - Update shipment record with driving_distance and driving_time.

18. **Add Google Sheets Node to Collect Shipment Information**  
    - Read full shipment record filtered by shipment_number to prepare data for email reply.

19. **Add OpenAI Chat Model Node for Reply Generation**  
    - Model: gpt-4o-mini  
    - Provide OpenAI API credentials.

20. **Add AI Agent Reply Node**  
    - Type: LangChain Agent  
    - Input text: Template listing shipment details with placeholders from collected shipment info and sender info from AI Agent Parser.  
    - System message: Defines role as logistics assistant crafting formal, polite shipment confirmation email in HTML, in English, no invented data, including company contact info.  
    - Connect to OpenAI Chat Model node.

21. **Add Gmail Node to Send Reply**  
    - Operation: Reply  
    - Message ID: From Gmail Trigger original email  
    - Message body: From AI Agent Reply output (HTML)  
    - Provide Gmail OAuth2 credentials.

22. **Connect all nodes as per dependencies described above.**

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Setup instructions for Gmail Trigger node including OAuth2 credentials.                                                                                                                                                                                                                                                       | https://docs.n8n.io/integrations/builtin/trigger-nodes/n8n-nodes-base.gmailtrigger                                |
| AI Agent node usage and prompt design tips with OpenAI GPT-4o-mini model.                                                                                                                                                                                                                                                      | https://docs.n8n.io/integrations/builtin/cluster-nodes/root-nodes/n8n-nodes-langchain.agent                      |
| Google Sheets node setup for appending, updating, and filtering rows with OAuth2 authentication.                                                                                                                                                                                                                              | https://docs.n8n.io/integrations/builtin/app-nodes/n8n-nodes-base.googlesheets                                  |
| Open Route API documentation and API key setup instructions.                                                                                                                                                                                                                                                                   | https://openrouteservice.org/dev/#/api-docs                                                                      |
| Stylish, professional email response formatting with HTML tags in AI prompt to ensure correct email display.                                                                                                                                                                                                                   | (Best practice advised in AI Agent Reply system prompt)                                                         |
| Video tutorial linked for overview and demonstration of similar workflow.                                                                                                                                                                                                                                                      | https://youtu.be/0InkUBOUMQQ                                                                                      |
| Branding and contact info for AI assistant in confirmation emails: LogiGreen Bot, Transportation Planner, LogiGreen Transportation, logigreenbot@gmail.com                                                                                                                                                                    | Used in AI Agent Reply system message.                                                                            |

---

**Disclaimer:**  
The text and workflow described here originate entirely from an automated n8n workflow designed for legal and public data processing. The workflow complies with all applicable content policies and contains no illegal or offensive elements.

---