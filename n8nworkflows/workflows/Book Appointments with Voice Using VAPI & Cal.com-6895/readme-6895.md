Book Appointments with Voice Using VAPI & Cal.com

https://n8nworkflows.xyz/workflows/book-appointments-with-voice-using-vapi---cal-com-6895


# Book Appointments with Voice Using VAPI & Cal.com

### 1. Workflow Overview

This workflow, titled **"Appointment Booking Voice Agent"**, automates booking and availability checking of appointments via voice commands processed through VAPI (Voice API) and Cal.com scheduling platform. It is designed to handle user intents received from a voice agent, interpret these intents, and then interact with Cal.com APIs to either check available slots or book an appointment.

The workflow is composed of the following logical blocks:

- **1.1 Input Reception:** Receives structured voice commands via a webhook from VAPI.
- **1.2 Preparation of Variables:** Extracts and sets key variables required for API calls (e.g., event type, username).
- **1.3 Intent Routing:** Routes the workflow based on user intent (check availability or book appointment).
- **1.4 Date-Time Extraction and Formatting:** Parses and formats spoken appointment date/time to fit Cal.com API requirements.
- **1.5 Check Availability Block:** Queries Cal.com for available time slots based on user input.
- **1.6 Book Appointment Block:** Sends booking requests to Cal.com with user details.
- **1.7 Response Handling:** Sends appropriate JSON responses back to VAPI for voice feedback.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

- **Overview:**  
  This block is the entry point where the workflow receives voice agent data via a webhook. The webhook expects POST requests containing structured user intent and entities.

- **Nodes Involved:**  
  - Webhook  
  - Set Variables

- **Node Details:**

  1. **Webhook**  
     - *Type:* Webhook (n8n-nodes-base.webhook)  
     - *Role:* Receives HTTP POST requests with voice commands from VAPI.  
     - *Configuration:*  
       - Disabled by default (likely used for testing or controlled activation).  
       - Path set to a unique webhook ID.  
       - HTTP method: POST.  
       - Response mode: uses response nodes downstream to reply.  
     - *Input:* External POST request containing structured JSON with voice intent (e.g., function names like "check-availability" or "book-appointment").  
     - *Output:* Raw JSON payload forwarded downstream.  
     - *Edge Cases:*  
       - Disabled state means no incoming requests processed.  
       - Malformed requests or missing expected fields may cause expression failures downstream.  

  2. **Set Variables**  
     - *Type:* Set (n8n-nodes-base.set)  
     - *Role:* Sets static variables required for Cal.com API calls, such as username, event type slug, and event type ID. These appear to be hardcoded.  
     - *Configuration:*  
       - username = "nabin-bhandari11"  
       - eventTypeSlug = "30 min"  
       - eventTypeId = "2964463" (likely an internal Cal.com event type identifier)  
     - *Input:* Output of Webhook node.  
     - *Output:* JSON enriched with these static variables.  
     - *Edge Cases:*  
       - Hardcoded values reduce flexibility; change requires workflow edit.  

---

#### 2.2 Preparation of Payload and Intent Routing

- **Overview:**  
  This block extracts the body from the webhook JSON, then routes the workflow based on the voice intent to either check availability or book an appointment.

- **Nodes Involved:**  
  - Prepare Payload Fields (Set)  
  - Switch

- **Node Details:**

  1. **Prepare Payload Fields**  
     - *Type:* Set  
     - *Role:* Extracts the entire body object from the webhook payload for easier access downstream.  
     - *Configuration:*  
       - Assigns `body = webhook JSON body`  
     - *Input:* Output from Set Variables node.  
     - *Output:* JSON with a `body` field containing the webhook message.  
     - *Edge Cases:*  
       - Assumes webhook payload structure is consistent.  

  2. **Switch**  
     - *Type:* Switch  
     - *Role:* Routes based on the function name extracted from the webhook payload, distinguishing between "check-availability" and "book-appointment" intents.  
     - *Configuration:*  
       - Two outputs named "Check Availability" and "Book Appointment".  
       - Conditions check:  
         - For "Check Availability": `body.message.toolCalls[0].function.name == "check-availability"` (case sensitive).  
         - For "Book Appointment": `body.message.toolWithToolCallList[0].function.name == "book-appointment"`.  
     - *Input:* Output from Prepare Payload Fields node.  
     - *Output:* Routes to two different branches accordingly.  
     - *Edge Cases:*  
       - If function names differ or are missing, no route will be matched, potentially halting workflow.  
       - Slight difference in JSON path for function name between conditions may cause routing issues if payload structure changes.  

---

#### 2.3 Date-Time Extraction and Formatting (Check Availability branch)

- **Overview:**  
  Parses spoken date/time from the voice command and formats it into start and end timestamps matching Cal.com API’s query parameters.

- **Nodes Involved:**  
  - Extract Start & End Time (Set)  
  - Check Availability (HTTP Request)  
  - Check Availability successful (Respond to Webhook)

- **Node Details:**

  1. **Extract Start & End Time**  
     - *Type:* Set  
     - *Role:* Extracts requested appointment start time and calculates an end time one day later (to define a time window).  
     - *Configuration:*  
       - `start` = `body.message.toolCalls[0].function.arguments.requestedappointment` (string)  
       - `end` = `body.message.toolCalls[0].function.arguments.requestedappointment.toDateTime().plus(1, 'days')` (adds 1 day using n8n date-time functions)  
     - *Input:* Output from Switch node on "Check Availability" route.  
     - *Output:* JSON with fields `" start"` and `" end"` (note the leading space in keys).  
     - *Edge Cases:*  
       - Assumes date string parsing succeeds; invalid or missing dates cause expression failures.  
       - The leading space in keys (`" start"`) is unconventional and could cause confusion in later steps.  

  2. **Check Availability**  
     - *Type:* HTTP Request  
     - *Role:* Sends a GET request to Cal.com API to fetch available slots within the specified start-end range for the event type.  
     - *Configuration:*  
       - URL: `https://api.cal.com/v2/slots`  
       - Query parameters:  
         - `start` = extracted start time from previous node  
         - `end` = extracted end time  
         - `eventTypeId` from Set Variables node  
       - Header: `cal-api-version: 2024-09-04`  
       - Authentication: HTTP Header Auth (custom header credential)  
     - *Input:* From Extract Start & End Time node.  
     - *Output:* API response with available slots.  
     - *Edge Cases:*  
       - Auth failure if credentials invalid.  
       - API downtime or rate limiting may cause HTTP errors.  
       - Malformed query parameters cause API errors.  

  3. **Check Availability successful**  
     - *Type:* Respond to Webhook  
     - *Role:* Sends JSON response back to VAPI confirming availability results.  
     - *Configuration:*  
       - Response body includes:  
         - `toolCallId` from webhook payload  
         - `result` containing the data field from API response (list of available slots)  
       - Responds with JSON.  
     - *Input:* Output from Check Availability node.  
     - *Output:* HTTP response to the original webhook request.  
     - *Edge Cases:*  
       - If API response is empty or malformed, may return empty or incorrect results.  

---

#### 2.4 Book Appointment Block

- **Overview:**  
  Sends a booking request to Cal.com based on user details and requested appointment date/time received from the voice agent.

- **Nodes Involved:**  
  - Book Appointment (HTTP Request)  
  - Booking SuccessFul (Respond to Webhook)

- **Node Details:**

  1. **Book Appointment**  
     - *Type:* HTTP Request  
     - *Role:* Sends a POST request to Cal.com bookings endpoint to create an appointment.  
     - *Configuration:*  
       - URL: `https://api.cal.com/v2/bookings`  
       - HTTP Method: POST  
       - JSON body includes:  
         - `attendee`: object with `language: "en"`, `name`, `timeZone`, `email` extracted from webhook message arguments  
         - `start`: requested appointment date/time from webhook payload  
         - `eventTypeId`: from Set Variables node  
       - Header: `cal-api-version: 2024-08-13`  
       - Authentication: HTTP Header Auth (using the same credential as availability check)  
     - *Input:* From Switch node route "Book Appointment".  
     - *Output:* API response confirming booking.  
     - *Edge Cases:*  
       - API errors if date/time invalid or already booked.  
       - Auth errors if credential invalid.  
       - Missing user info causes booking failure.  

  2. **Booking SuccessFul**  
     - *Type:* Respond to Webhook  
     - *Role:* Sends confirmation JSON back to VAPI indicating success of booking.  
     - *Configuration:*  
       - Response includes `toolCallId` from webhook payload and a success message.  
     - *Input:* Output of Book Appointment node.  
     - *Output:* HTTP response to webhook.  
     - *Edge Cases:*  
       - If booking API fails but this node is reached, may return false positives.  

---

#### 2.5 Sticky Notes Context

Sticky notes provide clarifying comments for the workflow:

- Entry point explanation (Webhook node): receives structured voice intents.
- Variable extraction clarifications.
- Intent routing explanation.
- Date/time formatting details.
- API call explanations for availability and booking.
- Response formatting for voice feedback.

---

### 3. Summary Table

| Node Name                  | Node Type           | Functional Role                                   | Input Node(s)            | Output Node(s)                 | Sticky Note                                                                                                 |
|----------------------------|---------------------|-------------------------------------------------|--------------------------|-------------------------------|-------------------------------------------------------------------------------------------------------------|
| Webhook                    | Webhook             | Entry point receiving voice agent requests      | -                        | Set Variables                  | Entry point from VAPI. This webhook receives user intent and entities from the voice agent in a structured format (e.g. action = "book" or "check"). |
| Set Variables              | Set                 | Sets static variables like username, eventType  | Webhook                  | Prepare Payload Fields         | Extracts variables like eventtypeid, eventtypeslug and username.                                            |
| Prepare Payload Fields     | Set                 | Extracts webhook body for easier access          | Set Variables             | Switch                        | Extracts the body from the webhook                                                                           |
| Switch                    | Switch              | Routes workflow based on user intent             | Prepare Payload Fields    | Extract Start & End Time (Check Availability), Book Appointment | Routes the voice intent: If the user said "Check availability", go to the check flow. If "Book appointment", go to booking flow. |
| Extract Start & End Time   | Set                 | Parses and formats date/time from voice input    | Switch                   | Check Availability            | Parses and formats spoken date/time info from the voice assistant to match Cal.com API's time range requirements. |
| Check Availability        | HTTP Request        | Queries Cal.com for available slots               | Extract Start & End Time  | Check Availability successful | Sends a GET request to Cal.com to fetch available time slots for the specified date range from the voice input. |
| Check Availability successful | Respond to Webhook | Returns availability results to voice agent      | Check Availability       | -                             | Returns a voice-ready response to VAPI (e.g. "There’s availability at 3 PM and 4 PM. Would you like to book?"). |
| Book Appointment          | HTTP Request        | Sends booking request to Cal.com                   | Switch                   | Booking SuccessFul            | Sends a POST request to Cal.com to book the appointment using parsed info from the voice conversation.       |
| Booking SuccessFul        | Respond to Webhook  | Returns confirmation of booking                    | Book Appointment         | -                             | Returns a spoken confirmation via VAPI (e.g. "Your appointment is confirmed for 3 PM on Tuesday")             |
| Sticky Note               | Sticky Note         | Comments/explanations                              | -                        | -                             | Various notes attached contextually to nodes                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: Use a unique identifier (e.g., "308f0eec-8459-4e6c-bf25-5cd9b2c31ad6")  
   - Response Mode: Use response nodes downstream  
   - Keep node enabled for production use  

2. **Create Set Variables Node:**  
   - Type: Set  
   - Add fields:  
     - `username` = "nabin-bhandari11"  
     - `eventTypeSlug` = "30 min"  
     - `eventTypeId` = "2964463"  
   - Connect Webhook → Set Variables  

3. **Create Prepare Payload Fields Node:**  
   - Type: Set  
   - Assign field:  
     - `body` = `{{$node["Webhook"].json["body"]}}` (copy webhook body)  
   - Connect Set Variables → Prepare Payload Fields  

4. **Create Switch Node:**  
   - Type: Switch  
   - Add two outputs:  
     - Output 1: "Check Availability" with condition:  
       `{{$json["body"]["message"]["toolCalls"][0]["function"]["name"]}} == "check-availability"`  
     - Output 2: "Book Appointment" with condition:  
       `{{$json["body"]["message"]["toolWithToolCallList"][0]["function"]["name"]}} == "book-appointment"`  
   - Connect Prepare Payload Fields → Switch  

5. **Create Extract Start & End Time Node:**  
   - Type: Set  
   - Add fields:  
     - ` start` = `{{$json["body"]["message"]["toolCalls"][0]["function"]["arguments"]["requestedappointment"]}}`  
     - ` end` = `{{$json["body"]["message"]["toolCalls"][0]["function"]["arguments"]["requestedappointment"].toDateTime().plus(1, "days")}}`  
   - Connect Switch (Check Availability output) → Extract Start & End Time  

6. **Create Check Availability HTTP Request Node:**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://api.cal.com/v2/slots`  
   - Query Parameters:  
     - `start` = `{{$json[" start"]}}`  
     - `end` = `{{$json[" end"]}}`  
     - `eventTypeId` = `{{$node["Set Variables"].json["eventTypeId"]}}`  
   - Headers: `cal-api-version: 2024-09-04`  
   - Authentication: HTTP Header Auth (configure with Cal.com API key)  
   - Connect Extract Start & End Time → Check Availability  

7. **Create Check Availability successful Node:**  
   - Type: Respond to Webhook  
   - Response Body (JSON):  
     ```json
     {
       "results": [
         {
           "toolCallId": "={{$json.body.message.toolCallList[0].id}}",
           "result": "={{$json.data}}"
         }
       ]
     }
     ```  
   - Connect Check Availability → Check Availability successful  

8. **Create Book Appointment HTTP Request Node:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.cal.com/v2/bookings`  
   - Body Content (JSON):  
     ```json
     {
       "attendee": {
         "language": "en",
         "name": "={{$json.body.message.toolCalls[0].function.arguments.Name}}",
         "timeZone": "={{$json.body.message.toolCalls[0].function.arguments.callerTimeZone}}",
         "email": "={{$json.body.message.toolCalls[0].function.arguments.Email}}"
       },
       "start": "={{$json.body.message.toolCallList[0].function.arguments.requestedappointmentdate}}",
       "eventTypeId": {{$node["Set Variables"].json["eventTypeId"]}}
     }
     ```  
   - Headers: `cal-api-version: 2024-08-13`  
   - Authentication: HTTP Header Auth (Cal.com API key)  
   - Connect Switch (Book Appointment output) → Book Appointment  

9. **Create Booking SuccessFul Respond to Webhook Node:**  
   - Type: Respond to Webhook  
   - Response Body (JSON):  
     ```json
     {
       "results": [
         {
           "toolCallId": "={{$json.body.message.toolCallList[0].id}}",
           "result": "Booking Successful"
         }
       ]
     }
     ```  
   - Connect Book Appointment → Booking SuccessFul  

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                                                                             |
|-----------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| The workflow integration relies on Cal.com’s API versions "2024-08-13" and "2024-09-04".       | Cal.com API documentation: https://cal.com/docs/api                                                           |
| Voice intents are structured as JSON with `toolCalls` or `toolWithToolCallList` arrays holding function names and arguments. | VAPI integration expects responses in JSON format with `toolCallId` matching the request for correlation.  |
| Hardcoded variables for eventTypeId and username require manual update for other event types. | To support multiple event types, extend the Set Variables node or add dynamic extraction from webhook data.|
| Authentication uses HTTP Header Auth credentials storing API keys securely in n8n credentials. | Credentials must be configured in n8n before running the workflow.                                         |
| Leading spaces in JSON keys (e.g. `" start"`) may cause confusion or errors in some nodes.    | Best practice is to avoid spaces in key names; consider renaming to `start` and `end` for clarity.          |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to content policies and does not contain any illegal, offensive, or protected elements. All manipulated data are legal and public.