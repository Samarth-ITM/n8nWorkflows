Voice-Based Appointment Booking System with ElevenLabs AI and Cal.com

https://n8nworkflows.xyz/workflows/voice-based-appointment-booking-system-with-elevenlabs-ai-and-cal-com-5670


# Voice-Based Appointment Booking System with ElevenLabs AI and Cal.com

---

### 1. Workflow Overview

This workflow implements a **Voice-Based Appointment Booking System** integrating ElevenLabs AI (implied by the title though not visible in provided nodes) and Cal.com scheduling API. Its core purpose is to receive appointment-related requests via a webhook, determine whether the request is to check available slots or to book an appointment, interact with Cal.com API accordingly, and respond back to the requester.

**Target use cases:**  
- Automated appointment slot availability checking  
- Automated appointment booking via REST API calls to Cal.com  
- Real-time interaction via HTTP webhook endpoints  

**Logical blocks:**  
- **1.1 Input Reception:** Receives incoming HTTP POST requests with appointment data.  
- **1.2 Request Type Decision:** Determines if the request is to check availability or to book an appointment.  
- **1.3 Cal.com Slot Checking:** Queries Cal.com API for available slots within a specified time window.  
- **1.4 Cal.com Appointment Booking:** Books an appointment using Cal.com API based on user input.  
- **1.5 Response Dispatch:** Sends back the API response to the original requester.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives external HTTP POST requests containing appointment-related JSON payloads. Acts as the entry point of the workflow.

- **Nodes Involved:**  
Webhook

- **Node Details:**  
  - **Webhook**  
    - Type: HTTP Webhook Trigger  
    - Configuration: Listens on path `appointment-webhook` for POST requests, responds asynchronously via a linked response node.  
    - Expressions: N/A (triggers on incoming HTTP requests)  
    - Input: External HTTP client  
    - Output: JSON payload from the request body (`$json.body`)  
    - Edge cases: malformed requests, missing or invalid JSON body, HTTP method mismatch  
    - Version-specific: Uses v2 webhook node with response mode set to `responseNode` for asynchronous response handling  

---

#### 1.2 Request Type Decision

- **Overview:**  
Determines if the incoming request is to check available slots or to book an appointment by inspecting the `tool` property in the request body.

- **Nodes Involved:**  
Check Is Request For Available Slot (If node)

- **Node Details:**  
  - **Check Is Request For Available Slot**  
    - Type: Conditional (If) node  
    - Configuration: Checks if `$json.body.tool` equals `"checkAvailableSlot"` (case-sensitive, strict type validation)  
    - Input: JSON body from Webhook  
    - Output: Two branches:  
      - True branch (tool = "checkAvailableSlot") routes to slot checking  
      - False branch routes to booking  
    - Edge cases: Missing `tool` property, unexpected values, case sensitivity could cause false negatives  
    - Version-specific: Using version 2.2 of If node for improved condition handling  

---

#### 1.3 Cal.com Slot Checking

- **Overview:**  
Queries Cal.com API to find available appointment slots between the requested start time and the end of the day (6 PM Asia/Kolkata timezone).

- **Nodes Involved:**  
Check available slot in Cal.com

- **Node Details:**  
  - **Check available slot in Cal.com**  
    - Type: HTTP Request (GET)  
    - Configuration:  
      - URL: `https://api.cal.com/v1/slots`  
      - Query parameters:  
        - `startTime` from `$json.body.startTime` (ISO string expected)  
        - `endTime` is dynamically computed as the same day with time set to 18:00:00 (6 PM) Asia/Kolkata timezone via DateTime expression  
        - `eventTypeId`: static placeholder `"event_type_id"` (should be replaced with actual event type ID)  
        - `timeZone`: `"Asia/Kolkata"`  
      - Authentication: Uses predefined Cal.com account credentials  
      - HTTP Method: GET (default)  
    - Input: JSON body from Webhook, filtered by If node  
    - Output: JSON response with available slots  
    - Edge cases:  
      - Invalid or missing startTime format  
      - API authentication failure or rate limits  
      - Network timeouts  
      - Placeholder `event_type_id` must be replaced with valid ID or request will fail  
    - Version-specific: HTTP Request node v4.2  

---

#### 1.4 Cal.com Appointment Booking

- **Overview:**  
Books a 30-minute appointment slot on Cal.com using provided user details and the specified start time.

- **Nodes Involved:**  
Book an Appointment

- **Node Details:**  
  - **Book an Appointment**  
    - Type: HTTP Request (POST)  
    - Configuration:  
      - URL: `https://api.cal.com/v1/bookings`  
      - Method: POST  
      - Body (JSON):  
        - `eventTypeId`: static `"event_type_id"` placeholder (must be replaced)  
        - `start`: from `$json.body.startTime` (ISO format)  
        - `end`: computed as start time plus 30 minutes via DateTime expression  
        - `responses`: user `name` and `email` from request body  
        - `timeZone`: `"Asia/Kolkata"`  
        - `language`: `"en"`  
        - `title`: fixed `"Test"` (can be customized)  
        - `metadata`: empty object (optional)  
      - Authentication: Predefined Cal.com credentials  
    - Input: JSON body from Webhook, routed when tool ≠ `"checkAvailableSlot"`  
    - Output: JSON booking confirmation or error  
    - Edge cases:  
      - Invalid or missing user information (name, email)  
      - Overlapping or unavailable slots  
      - API errors due to invalid eventTypeId or malformed request  
      - Timezone inconsistencies  
    - Version-specific: HTTP Request node v4.2  

---

#### 1.5 Response Dispatch

- **Overview:**  
Sends the JSON response from Cal.com APIs back to the original HTTP requester.

- **Nodes Involved:**  
Respond to Webhook

- **Node Details:**  
  - **Respond to Webhook**  
    - Type: Respond to Webhook node  
    - Configuration: Default options, sends the output of the preceding node as HTTP response  
    - Input: Receives output from either slot checking or booking nodes  
    - Output: HTTP response to client  
    - Edge cases:  
      - If previous node errors out, response may be empty or error message  
      - Needs to handle different response structures (slots list vs booking confirmation) gracefully  
    - Version-specific: v1.2  

---

### 3. Summary Table

| Node Name                      | Node Type             | Functional Role                  | Input Node(s)             | Output Node(s)               | Sticky Note                                         |
|-------------------------------|-----------------------|--------------------------------|---------------------------|-----------------------------|-----------------------------------------------------|
| Webhook                       | Webhook Trigger       | Entry point, receives requests | (HTTP client external)    | Check Is Request For Available Slot |                                                     |
| Check Is Request For Available Slot | If Node              | Route request by tool type      | Webhook                   | Check available slot in Cal.com (true branch), Book an Appointment (false branch) |                                                     |
| Check available slot in Cal.com | HTTP Request          | Query available slots           | Check Is Request For Available Slot | Respond to Webhook          |                                                     |
| Book an Appointment           | HTTP Request          | Book appointment                | Check Is Request For Available Slot | Respond to Webhook          |                                                     |
| Respond to Webhook            | Respond to Webhook    | Send response to requester      | Check available slot in Cal.com, Book an Appointment | (HTTP client external)       |                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `appointment-webhook`  
   - Response Mode: `responseNode` (to send response asynchronously)  

2. **Create If Node: Check Is Request For Available Slot**  
   - Type: If  
   - Condition: `$json.body.tool` equals `"checkAvailableSlot"` (case-sensitive, strict)  
   - Connect Webhook node output to this node input  

3. **Create HTTP Request Node: Check available slot in Cal.com**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.cal.com/v1/slots`  
   - Query Parameters:  
     - `startTime`: `={{ $json.body.startTime }}`  
     - `endTime`: computed using:  
       ```javascript
       =DateTime.fromISO($json.body.startTime, { zone: 'Asia/Kolkata' })
         .set({ hour: 18, minute: 0, second: 0 })
         .format("yyyy-MM-dd'T'HH:mm:ssZZ")
       ```  
     - `eventTypeId`: Replace `"event_type_id"` with your actual Cal.com event type ID  
     - `timeZone`: `"Asia/Kolkata"`  
   - Authentication: Use a saved Cal.com credential (OAuth2 or API key)  
   - Connect If node true output to this node input  

4. **Create HTTP Request Node: Book an Appointment**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.cal.com/v1/bookings`  
   - Body (JSON):  
     ```json
     {
       "eventTypeId": "event_type_id",
       "start": "{{ $json.body.startTime }}",
       "end": "{{ DateTime.fromISO($json.body.startTime).plus({ minutes: 30 }).format(\"yyyy-MM-dd'T'HH:mm:ssZZ\") }}",
       "responses": {
         "name": "{{ $json.body.name }}",
         "email": "{{ $json.body.email }}"
       },
       "timeZone": "Asia/Kolkata",
       "language": "en",
       "title": "Test",
       "metadata": {}
     }
     ```  
   - Authentication: Use same Cal.com credentials as above  
   - Connect If node false output to this node input  

5. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Connect outputs of both HTTP Request nodes (slot checking and booking) to this node input  
   - This node sends the API response back to the HTTP client  

6. **Set up Credentials for Cal.com API**  
   - Create new credentials in n8n for Cal.com API (API key or OAuth2)  
   - Assign this credential to both HTTP Request nodes interacting with Cal.com  

7. **Save and Activate Workflow**  
   - Ensure the webhook URL is accessible externally  
   - Test with POST requests containing JSON bodies that have at minimum:  
     - `tool` property with values `"checkAvailableSlot"` or others for booking  
     - `startTime` in ISO format  
     - For booking, include `name` and `email` fields  

---

### 5. General Notes & Resources

| Note Content                                                                                                  | Context or Link                            |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------|
| The placeholder `"event_type_id"` must be replaced with a valid event type ID from your Cal.com account.      | Cal.com API documentation                   |
| Timezone is fixed to `"Asia/Kolkata"`; adjust if your use case requires a different timezone.                  | Consider n8n DateTime node for timezone adjustments |
| The workflow is inactive (`active: false`) in current export; remember to activate before production use.     | n8n workflow settings                        |
| Webhook node uses `responseNode` mode for async response handling.                                            | n8n Webhook node docs                        |
| This workflow assumes valid and well-formed JSON requests; consider adding validation or error handling nodes. | Best practice for robust API endpoints      |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.

---