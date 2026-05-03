Create a Calendly Availability API Endpoint for Real-time Scheduling Data

https://n8nworkflows.xyz/workflows/create-a-calendly-availability-api-endpoint-for-real-time-scheduling-data-11254


# Create a Calendly Availability API Endpoint for Real-time Scheduling Data

### 1. Workflow Overview

This workflow, named **Calendly Availability Checker**, is designed to provide a real-time API endpoint for retrieving Calendly scheduling availability. It enables users or applications to query available time slots for specific Calendly event types within a configurable future date range. The workflow authenticates with the Calendly API, retrieves user and event type data, fetches available booking times, and formats the response for easy consumption.

**Target Use Cases:**
- Displaying available scheduling slots on websites or apps
- Pre-checking availability before proposing meeting times
- Building custom booking interfaces integrated with Calendly
- Integrating availability checks into chatbots or other conversational agents

**Logical Blocks:**

- **1.1 Input Reception and Configuration:** Receives webhook requests with optional parameters for event type and date range, sets configuration variables.
- **1.2 User and Event Data Retrieval:** Authenticates with Calendly and fetches user information and available event types.
- **1.3 Event Type Selection:** Determines which event type to check based on input or default.
- **1.4 Availability Query:** Requests real-time available times for the selected event type within the specified date range.
- **1.5 Response Formatting and Output:** Formats availability data into a structured JSON response and returns it to the requester.
- **1.6 Workflow Metadata and Setup Notes:** Sticky notes provide overview, trigger instructions, setup guide, and response format reference.

---

### 2. Block-by-Block Analysis

---

#### Block 1.1: Input Reception and Configuration

**Overview:**  
Handles incoming API requests via webhook, extracts optional parameters from the request body to configure which event type to check and how many days ahead to look for availability.

**Nodes Involved:**  
- Webhook Trigger  
- Set Configuration  

**Node Details:**

- **Webhook Trigger**  
  - *Type:* Webhook (HTTP POST)  
  - *Role:* Entry point accepting POST requests on path `/check-calendly-availability`.  
  - *Configuration:* Responds with the output of the final response node. Expects optional JSON body with `event_type_uri` and `days_ahead`.  
  - *Input/Output:* No input; outputs request data to Set Configuration.  
  - *Edge Cases:* Missing or malformed body; no event_type_uri provided (handled downstream).  
  - *Notes:* Sticky note describes request body format and default behavior if no event_type_uri is provided.

- **Set Configuration**  
  - *Type:* Set  
  - *Role:* Extracts and sets workflow variables `requested_event_type` (string) and `days_ahead` (number) from webhook request body or defaults.  
  - *Key Expressions:*  
    - `requested_event_type` = `{{$json.body?.event_type_uri || ''}}`  
    - `days_ahead` = `{{$json.body?.days_ahead || 7}}`  
  - *Input:* Output from Webhook Trigger  
  - *Output:* Configuration data for next node  
  - *Edge Cases:* Missing parameters default to empty string or 7 days.

---

#### Block 1.2: User and Event Data Retrieval

**Overview:**  
Authenticates with Calendly API, retrieves current user details, then fetches all active event types associated with the user.

**Nodes Involved:**  
- Get Current User  
- Extract User Info  
- Get Event Types  

**Node Details:**

- **Get Current User**  
  - *Type:* HTTP Request  
  - *Role:* GET request to `https://api.calendly.com/users/me` to retrieve user profile.  
  - *Authentication:* Uses Calendly OAuth2 credential.  
  - *On Error:* Continues execution to avoid blocking on API errors.  
  - *Input:* Output from Set Configuration  
  - *Output:* User data JSON to Extract User Info  
  - *Edge Cases:* API auth failure, rate limit, network issues.

- **Extract User Info**  
  - *Type:* Set  
  - *Role:* Parses relevant user fields (`uri`, `name`, `email`, `current_organization`, `scheduling_url`) from API response.  
  - *Key Expressions:*  
    - `user_uri` = `{{$json.resource?.uri}}`  
    - `user_name` = `{{$json.resource?.name}}`  
    - `user_email` = `{{$json.resource?.email}}`  
    - `organization_uri` = `{{$json.resource?.current_organization}}`  
    - `scheduling_url` = `{{$json.resource?.scheduling_url}}`  
  - *Input:* Output from Get Current User  
  - *Output:* User info to Get Event Types  
  - *Edge Cases:* Missing fields due to API errors or data changes.

- **Get Event Types**  
  - *Type:* HTTP Request  
  - *Role:* GET request to `https://api.calendly.com/event_types` with query parameters `user` (user URI) and `active=true` to get event types.  
  - *Authentication:* Calendly OAuth2 credential.  
  - *On Error:* Continues execution to avoid blocking downstream.  
  - *Input:* Output from Extract User Info  
  - *Output:* Event types JSON to Select Event Type  
  - *Edge Cases:* Empty event types list, API errors.

---

#### Block 1.3: Event Type Selection

**Overview:**  
Determines which event type to use for availability checks, either from the requested input or defaults to the first active event type.

**Nodes Involved:**  
- Select Event Type  

**Node Details:**

- **Select Event Type**  
  - *Type:* Set  
  - *Role:*  
    - Counts event types returned  
    - Creates a simplified list of event types with key info (URI, name, duration, slug)  
    - Selects either requested event type URI or first event type URI  
    - Sets event name and duration for the selected event type  
  - *Key Expressions:*  
    - `event_types_count` = `{{$json.collection?.length || 0}}`  
    - `event_types_list` = mapping over event types for summary  
    - `selected_event_type_uri` = `{{ $('Set Configuration').item.json.requested_event_type || ($json.collection && $json.collection.length > 0 ? $json.collection[0].uri : '') }}`  
    - `selected_event_type_name` and `selected_event_duration` correspond to the first event type or fallback values  
  - *Input:* Output from Get Event Types  
  - *Output:* Selected event type info to Get Available Times  
  - *Edge Cases:* No event types available, requested event type URI not present.

---

#### Block 1.4: Availability Query

**Overview:**  
Queries Calendly API for available times for the selected event type within the specified date range (default 7 days).

**Nodes Involved:**  
- Get Available Times  

**Node Details:**

- **Get Available Times**  
  - *Type:* HTTP Request  
  - *Role:* GET request to `https://api.calendly.com/event_type_available_times` with parameters:  
    - `event_type` (URI)  
    - `start_time` (ISO string for start date; set to tomorrow)  
    - `end_time` (ISO string for end date; start + `days_ahead`)  
  - *Authentication:* Calendly OAuth2 credential  
  - *On Error:* Continues execution on failure  
  - *Key Expressions:*  
    - `start_time` = tomorrow's date in ISO format  
    - `end_time` = date after `days_ahead` days in ISO format  
  - *Input:* Output from Select Event Type and Set Configuration (for days_ahead)  
  - *Output:* Available time slots to Format Availability  
  - *Edge Cases:* No slots available, API errors, invalid dates.

---

#### Block 1.5: Response Formatting and Output

**Overview:**  
Transforms raw availability data into a structured and human-readable JSON format, then returns it to the API caller.

**Nodes Involved:**  
- Format Availability  
- Respond with Availability  

**Node Details:**

- **Format Availability**  
  - *Type:* Set  
  - *Role:*  
    - Calculates total slots and availability boolean  
    - Extracts next available slot and formats it for display  
    - Prepares a list of up to 20 available slots with formatted date/time and booking URLs  
    - Groups slots by day with readable keys and times  
  - *Key Expressions:*  
    - `total_slots_available` = length of slots array  
    - `has_availability` = boolean whether slots exist  
    - `next_available_slot` = ISO string of earliest slot or null  
    - `next_available_formatted` = localized string of earliest slot or fallback text  
    - `available_slots` = array of slot objects with start time, formatted string, and booking URL  
    - `slots_by_day` = object grouping slots by day with time and URL  
  - *Input:* Output from Get Available Times  
  - *Output:* Formatted data to Respond with Availability  
  - *Edge Cases:* Empty availability, malformed slot data.

- **Respond with Availability**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends HTTP 200 response with comprehensive JSON including:  
    - User info (name, email, scheduling URL)  
    - Selected event type info (name, duration, URI)  
    - Availability summary and detailed slots  
    - Timestamp of check  
  - *Response Headers:* Content-Type application/json  
  - *Input:* Output from Format Availability and earlier nodes via expressions  
  - *Output:* HTTP response to the original webhook request  
  - *Edge Cases:* Expression errors in response construction.

---

#### Block 1.6: Workflow Metadata and Documentation

**Overview:**  
Sticky notes providing human-readable information for workflow understanding, setup, triggers, and response format.

**Nodes Involved:**  
- 📋 WORKFLOW OVERVIEW (sticky note)  
- ⚙️ SETUP GUIDE (sticky note)  
- 🎯 TRIGGERS (sticky note)  
- 📤 RESPONSE FORMAT (sticky note)  

**Node Details:**

- Sticky notes contain setup instructions, example request body, trigger information, and JSON response schema.  
- Positioned for easy reference alongside workflow nodes.

---

### 3. Summary Table

| Node Name               | Node Type              | Functional Role                                | Input Node(s)           | Output Node(s)           | Sticky Note                                                                                                  |
|-------------------------|------------------------|------------------------------------------------|-------------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| 📋 WORKFLOW OVERVIEW    | Sticky Note            | Describes workflow purpose and use cases       |                         |                          | ## 🗓️ Calendly Availability Checker ... (overview and triggers description)                                 |
| Webhook Trigger          | Webhook                | Entry point receiving POST requests             |                         | Set Configuration        | POST /check-calendly-availability Optional body: { "event_type_uri": "..." }                                 |
| Set Configuration        | Set                    | Extracts parameters from webhook request body   | Webhook Trigger          | Get Current User         |                                                                                                              |
| Get Current User         | HTTP Request           | Gets authenticated user info from Calendly API | Set Configuration        | Extract User Info        |                                                                                                              |
| Extract User Info        | Set                    | Parses user data from API response               | Get Current User         | Get Event Types          |                                                                                                              |
| Get Event Types          | HTTP Request           | Retrieves all active event types for user        | Extract User Info        | Select Event Type        |                                                                                                              |
| Select Event Type        | Set                    | Selects requested or default event type          | Get Event Types          | Get Available Times      |                                                                                                              |
| Get Available Times      | HTTP Request           | Gets available booking slots for event type      | Select Event Type        | Format Availability      |                                                                                                              |
| Format Availability      | Set                    | Formats availability data for API response       | Get Available Times      | Respond with Availability|                                                                                                              |
| Respond with Availability| Respond to Webhook     | Sends JSON response with availability data       | Format Availability      |                          |                                                                                                              |
| ⚙️ SETUP GUIDE           | Sticky Note            | Setup instructions for credentials and workflow |                         |                          | ## ⚙️ Setup Instructions ... (Calendly credential setup and workflow config)                                 |
| 🎯 TRIGGERS              | Sticky Note            | Request body format and trigger explanation      |                         |                          | ## 🎯 Triggers ... (JSON body example and defaults)                                                          |
| 📤 RESPONSE FORMAT       | Sticky Note            | Example JSON response format                      |                         |                          | ## 📤 Response Format ... (sample formatted JSON response)                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Trigger Node:**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `check-calendly-availability`  
   - Response Mode: Use Respond Node output  
   - Notes: Accepts optional JSON body with `event_type_uri` (string) and `days_ahead` (number).  

2. **Create Set Node "Set Configuration":**  
   - Extract and assign:  
     - `requested_event_type` = `{{$json.body?.event_type_uri || ''}}`  
     - `days_ahead` = `{{$json.body?.days_ahead || 7}}`  
   - Connect Webhook Trigger → Set Configuration.

3. **Create HTTP Request Node "Get Current User":**  
   - Method: GET  
   - URL: `https://api.calendly.com/users/me`  
   - Authentication: Calendly OAuth2 API credential (configure OAuth2 in n8n with Calendly)  
   - On Error: Continue execution  
   - Connect Set Configuration → Get Current User.

4. **Create Set Node "Extract User Info":**  
   - Assign:  
     - `user_uri` = `{{$json.resource?.uri}}`  
     - `user_name` = `{{$json.resource?.name}}`  
     - `user_email` = `{{$json.resource?.email}}`  
     - `organization_uri` = `{{$json.resource?.current_organization}}`  
     - `scheduling_url` = `{{$json.resource?.scheduling_url}}`  
   - Include other fields  
   - Connect Get Current User → Extract User Info.

5. **Create HTTP Request Node "Get Event Types":**  
   - Method: GET  
   - URL: `https://api.calendly.com/event_types`  
   - Query Parameters:  
     - `user` = `{{$json.user_uri}}`  
     - `active` = `true`  
   - Authentication: Calendly OAuth2 API credential  
   - On Error: Continue execution  
   - Connect Extract User Info → Get Event Types.

6. **Create Set Node "Select Event Type":**  
   - Assign:  
     - `event_types_count` = `{{$json.collection?.length || 0}}`  
     - `event_types_list` = map over event types to extract `uri`, `name`, `duration`, `slug`  
     - `selected_event_type_uri` = `{{ $('Set Configuration').item.json.requested_event_type || ($json.collection && $json.collection.length > 0 ? $json.collection[0].uri : '') }}`  
     - `selected_event_type_name` and `selected_event_duration` default to first event type or fallback string/number  
   - Connect Get Event Types → Select Event Type.

7. **Create HTTP Request Node "Get Available Times":**  
   - Method: GET  
   - URL: `https://api.calendly.com/event_type_available_times`  
   - Query Parameters:  
     - `event_type` = `{{$json.selected_event_type_uri}}`  
     - `start_time` = ISO string for tomorrow's date: `new Date(Date.now() + 1 * 24 * 60 * 60 * 1000).toISOString()`  
     - `end_time` = ISO string for `days_ahead` days after today: `new Date(Date.now() + ($('Set Configuration').item.json.days_ahead || 7) * 24 * 60 * 60 * 1000).toISOString()`  
   - Authentication: Calendly OAuth2 API credential  
   - On Error: Continue execution  
   - Connect Select Event Type → Get Available Times.

8. **Create Set Node "Format Availability":**  
   - Assign:  
     - `total_slots_available` = length of slots array  
     - `has_availability` = boolean if slots exist  
     - `next_available_slot` = earliest slot ISO string or null  
     - `next_available_formatted` = localized formatted string or fallback text  
     - `available_slots` = array slice (up to 20) with formatted start time and booking URL  
     - `slots_by_day` = object grouping slots by day (weekday, month, day) and listing times and URLs  
   - Connect Get Available Times → Format Availability.

9. **Create Respond to Webhook Node "Respond with Availability":**  
   - Response Code: 200  
   - Response Headers: `Content-Type: application/json`  
   - Response Body (JSON): Construct an object with:  
     - `success: true`  
     - `user`: name, email, scheduling_url from Extract User Info  
     - `event_type`: name, duration_minutes, uri from Select Event Type  
     - `availability`: has_slots, total_slots, next_available (formatted), next_available_iso  
     - `slots`: JSON stringified `available_slots`  
     - `slots_by_day`: JSON stringified `slots_by_day`  
     - `all_event_types`: JSON stringified `event_types_list`  
     - `checked_at`: current ISO timestamp  
   - Connect Format Availability → Respond with Availability.

10. **Add Sticky Notes for Documentation:**  
    - Overview of workflow and use cases near start nodes  
    - Setup guide with Calendly OAuth2 credential instructions near end nodes  
    - Trigger instructions near Webhook node  
    - Response format example near Respond node  

11. **Set Workflow Execution Order:**  
    - Follow connections as above for sequential execution.

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                               |
|-----------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------|
| For Calendly API access, create a Personal Access Token or OAuth2 app at https://calendly.com/integrations       | Setup Guide sticky note                                                                                       |
| The webhook expects POST requests with optional JSON body containing `"event_type_uri"` and `"days_ahead"`       | Trigger sticky note                                                                                           |
| Default `days_ahead` is 7 days if not specified in request                                                      | Trigger sticky note                                                                                           |
| The response JSON includes comprehensive user, event type, and availability data suitable for frontend use      | Response Format sticky note                                                                                   |
| OAuth2 credentials must be configured in n8n with Calendly OAuth2 API scope                                      | Setup Guide and HTTP Request nodes configuration                                                             |
| Error handling: HTTP requests continue workflow on error to provide best-effort responses                        | HTTP Request nodes configured with `onError` to continue execution                                           |
| Timezone handling: dates are ISO formatted and localized display uses `en-US` locale                             | Format Availability node                                                                                      |
| This workflow can be triggered manually, on schedule, or via webhook for flexible integration                   | Workflow Overview sticky note                                                                                 |

---

**Disclaimer:** The provided content is exclusively derived from an n8n automated workflow. It strictly complies with all current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.