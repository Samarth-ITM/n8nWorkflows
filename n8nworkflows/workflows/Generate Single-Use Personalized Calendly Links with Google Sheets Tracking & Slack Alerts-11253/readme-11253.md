Generate Single-Use Personalized Calendly Links with Google Sheets Tracking & Slack Alerts

https://n8nworkflows.xyz/workflows/generate-single-use-personalized-calendly-links-with-google-sheets-tracking---slack-alerts-11253


# Generate Single-Use Personalized Calendly Links with Google Sheets Tracking & Slack Alerts

### 1. Workflow Overview

This workflow, **Calendly Booking Link Generator**, automates the creation of personalized, single-use booking links through the Calendly API. It is designed for scenarios where direct booking via Calendly's API is not supported, enabling users to generate unique booking URLs pre-filled with invitee information. These links automatically expire after a single booking or 90 days, and their usage is tracked and optionally notified via Slack.

**Target Use Cases:**
- CRM automation workflows that require scheduling links personalized for each contact.
- Sales outreach automation with individual booking links per prospect.
- Customer support scheduling with tracked invitations.
- Automated follow-ups where link expiration and tracking are essential.

**Logical Blocks:**

- **1.1 Input Reception:** Receives HTTP POST requests with recipient data.
- **1.2 Configuration Extraction:** Parses and sets recipient and event data.
- **1.3 Calendly Authentication & User Info:** Authenticates and fetches the current Calendly user.
- **1.4 Event Type Retrieval & Selection:** Fetches active event types and selects the requested or default event.
- **1.5 Single-Use Link Creation:** Generates a single-use scheduling link via Calendly API.
- **1.6 Link Personalization:** Appends recipient-specific query parameters to the generated link.
- **1.7 Logging & Notifications:** Logs link data into Google Sheets and optionally sends Slack notifications.
- **1.8 Webhook Response:** Returns the personalized booking link and related metadata to the requester.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

- **Overview:**  
  Captures incoming HTTP POST requests containing recipient and optional event type data.

- **Nodes Involved:**  
  - Webhook Trigger  
  - 🎯 INPUT FORMAT (Sticky Note)

- **Node Details:**  

  - **Webhook Trigger**  
    - *Type:* Webhook  
    - *Role:* Entry point accepting POST requests at `/generate-calendly-link`.  
    - *Configuration:* HTTP POST method, response mode set to wait for response node.  
    - *Input:* HTTP POST body with JSON containing `name`, `email`, and optional `event_type_uri`.  
    - *Output:* Passes raw request data downstream.  
    - *Edge Cases:* Missing or malformed JSON payload; invalid email format not validated here.

  - **🎯 INPUT FORMAT**  
    - *Type:* Sticky Note  
    - *Role:* Documentation of expected webhook input format.  
    - *Content:* Specifies JSON body structure and fallback behavior if `event_type_uri` is omitted.

---

#### 1.2 Configuration Extraction

- **Overview:**  
  Extracts and sets recipient name, email, requested event type URI, and UTM source from the webhook payload into workflow variables.

- **Nodes Involved:**  
  - Set Configuration

- **Node Details:**  

  - **Set Configuration**  
    - *Type:* Set  
    - *Role:* Maps incoming JSON body fields to internal variables with defaults.  
    - *Configuration:*  
      - `recipient_email` = `body.email` or `test@example.com`  
      - `recipient_name` = `body.name` or `Test User`  
      - `requested_event_type` = `body.event_type_uri` or empty string  
      - `utm_source` = `body.utm_source` or `'n8n'`  
    - *Input:* Raw webhook JSON  
    - *Output:* Structured JSON with extracted variables.  
    - *Edge Cases:* Defaults ensure workflow continues even with missing data.

---

#### 1.3 Calendly Authentication & User Info

- **Overview:**  
  Authenticates via OAuth2 to Calendly API and fetches current user details to identify the owner for event types.

- **Nodes Involved:**  
  - Get Current User  
  - Extract User

- **Node Details:**  

  - **Get Current User**  
    - *Type:* HTTP Request  
    - *Role:* Calls `GET https://api.calendly.com/users/me` to retrieve user info.  
    - *Authentication:* Uses Calendly OAuth2 credentials.  
    - *Error Handling:* Continues workflow even if API call fails (onError=continue).  
    - *Output:* User resource JSON.  

  - **Extract User**  
    - *Type:* Set  
    - *Role:* Extracts `user_uri` and `user_name` from API response for downstream queries.  
    - *Input:* Response from Get Current User.  
    - *Output:* JSON containing `user_uri` and `user_name`, plus other fields passed through.  
    - *Edge Cases:* If API response lacks expected fields, `user_uri` may be undefined, affecting next steps.

---

#### 1.4 Event Type Retrieval & Selection

- **Overview:**  
  Retrieves active event types for the user and selects either the requested event type or defaults to the first active one.

- **Nodes Involved:**  
  - Get Event Types  
  - Select Event Type

- **Node Details:**  

  - **Get Event Types**  
    - *Type:* HTTP Request  
    - *Role:* Calls `GET https://api.calendly.com/event_types?user={user_uri}&active=true` to get event types.  
    - *Authentication:* Calendly OAuth2.  
    - *Error Handling:* Continues on error.  
    - *Input:* Uses `user_uri` from Extract User node.  
    - *Output:* Collection of event types in `collection` array.  

  - **Select Event Type**  
    - *Type:* Set  
    - *Role:* Determines which event type to use: requested via payload or first active event.  
    - *Logic:*  
      - `selected_event_type_uri` = requested_event_type OR first event type URI from collection  
      - `selected_event_type_name` = first event name or empty string  
      - `selected_event_duration` = first event duration or 30 minutes default  
    - *Edge Cases:* If no event types available, defaults to empty strings and 30 minutes duration, which might cause downstream API errors.

---

#### 1.5 Single-Use Link Creation

- **Overview:**  
  Creates a single-use scheduling link limited to one booking for the selected event type.

- **Nodes Involved:**  
  - Create Single-Use Link

- **Node Details:**  

  - **Create Single-Use Link**  
    - *Type:* HTTP Request  
    - *Role:* Calls `POST https://api.calendly.com/scheduling_links` with JSON body specifying:  
      - `max_event_count`: 1 (single-use)  
      - `owner`: selected event type URI  
      - `owner_type`: `"EventType"`  
    - *Authentication:* Calendly OAuth2.  
    - *Error Handling:* Continues on error (may return incomplete or error response).  
    - *Output:* Scheduling link resource including `booking_url`.  
    - *Edge Cases:* If event type URI invalid or API error, no valid link generated.

---

#### 1.6 Link Personalization

- **Overview:**  
  Constructs a personalized booking URL by appending query parameters to pre-fill recipient info and track the source.

- **Nodes Involved:**  
  - Build Personalized Link

- **Node Details:**  

  - **Build Personalized Link**  
    - *Type:* Set  
    - *Role:* Creates variables:  
      - `base_booking_url` from API response  
      - `personalized_booking_url` = base URL + query parameters:  
        - `name` (URL encoded recipient name)  
        - `email` (URL encoded recipient email)  
        - `utm_source`  
      - Includes metadata like `event_type_name`, `event_duration`, `link_created_at`.  
    - *Edge Cases:* If API response lacks `booking_url`, personalized URL is invalid.

---

#### 1.7 Logging & Notifications

- **Overview:**  
  Logs each generated personalized link to Google Sheets for tracking and optionally sends a Slack notification.

- **Nodes Involved:**  
  - Log to Google Sheets  
  - Notify via Slack

- **Node Details:**  

  - **Log to Google Sheets**  
    - *Type:* Google Sheets node  
    - *Role:* Appends a new row with link and recipient info to a configured spreadsheet.  
    - *Sheet Headers:* Name, Email, Link, Event, Created, Duration, Status  
    - *Credentials:* Google Sheets OAuth2  
    - *Error Handling:* Continues on error (logging failure does not halt workflow).  
    - *Edge Cases:* Missing or invalid Google Sheets credentials or document ID; API throttling.

  - **Notify via Slack**  
    - *Type:* Slack node  
    - *Role:* Posts a message to a Slack channel with booking link details.  
    - *Message:* Includes recipient name/email, event name/duration, and clickable booking link.  
    - *Credentials:* Slack API OAuth token  
    - *Error Handling:* Continues on error (notification failure does not halt workflow).  
    - *Edge Cases:* Invalid Slack credentials, missing channel ID, Slack API rate limits.

---

#### 1.8 Webhook Response

- **Overview:**  
  Sends back a JSON response to the original webhook request containing the personalized booking link and metadata.

- **Nodes Involved:**  
  - Respond to Webhook  
  - 📤 RESPONSE (Sticky Note)

- **Node Details:**  

  - **Respond to Webhook**  
    - *Type:* Respond to Webhook  
    - *Role:* Returns HTTP 200 with JSON:  
      - `success`: true  
      - `booking_url`: personalized link  
      - `base_url`  
      - `recipient` info  
      - `event` info (name, duration)  
      - `created_at` timestamp  
      - `expires` info (single-use or 90 days)  
    - *Input:* Data from Build Personalized Link and enriched fields.  
    - *Output:* HTTP response to client.  

  - **📤 RESPONSE**  
    - *Type:* Sticky Note  
    - *Role:* Documents the JSON response format for clarity.

---

### 3. Summary Table

| Node Name             | Node Type            | Functional Role                        | Input Node(s)          | Output Node(s)            | Sticky Note                                                  |
|-----------------------|----------------------|-------------------------------------|-----------------------|---------------------------|--------------------------------------------------------------|
| 📋 WORKFLOW OVERVIEW  | Sticky Note          | Overview and purpose documentation   | —                     | —                         | Explanation of workflow purpose, use cases, and logic blocks |
| ⚙️ SETUP GUIDE         | Sticky Note          | Setup instructions for credentials   | —                     | —                         | Step-by-step setup instructions for Calendly, Google Sheets, Slack |
| 🎯 INPUT FORMAT        | Sticky Note          | Input format documentation           | —                     | —                         | Defines expected webhook POST body format                    |
| Webhook Trigger       | Webhook              | Entry point receiving POST data      | —                     | Set Configuration          | POST with recipient details                                  |
| Set Configuration     | Set                  | Extracts recipient and event info    | Webhook Trigger       | Get Current User           | Extract recipient info from request                          |
| Get Current User      | HTTP Request         | Fetch Calendly user info             | Set Configuration     | Extract User               | GET /users/me Calendly API                                   |
| Extract User          | Set                  | Extracts user URI and name           | Get Current User      | Get Event Types            | —                                                            |
| Get Event Types       | HTTP Request         | Fetch active event types             | Extract User          | Select Event Type          | GET /event_types?user=…&active=true                          |
| Select Event Type     | Set                  | Selects requested or default event   | Get Event Types       | Create Single-Use Link     | Pick requested or first event type                           |
| Create Single-Use Link| HTTP Request         | Creates single-use booking link      | Select Event Type     | Build Personalized Link    | POST /scheduling_links with max_event_count=1               |
| Build Personalized Link| Set                  | Adds prefill params to booking URL   | Create Single-Use Link| Log to Google Sheets       | Add prefill params to URL                                    |
| Log to Google Sheets  | Google Sheets        | Logs generated links                 | Build Personalized Link| Notify via Slack           | Track generated links                                        |
| Notify via Slack      | Slack                | Sends Slack notification             | Log to Google Sheets  | Respond to Webhook         | Optional notification                                        |
| Respond to Webhook    | Respond to Webhook   | Returns JSON response to caller      | Notify via Slack      | —                         | Return the generated link                                   |
| 📤 RESPONSE           | Sticky Note          | Response format documentation        | —                     | —                         | Shows JSON response example                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Webhook Trigger node:**
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `/generate-calendly-link`  
   - Response Mode: Wait for response node

2. **Add a Set node named "Set Configuration":**
   - Extract from webhook body:  
     - `recipient_email` = `{{$json.body.email || 'test@example.com'}}`  
     - `recipient_name` = `{{$json.body.name || 'Test User'}}`  
     - `requested_event_type` = `{{$json.body.event_type_uri || ''}}`  
     - `utm_source` = `{{$json.body.utm_source || 'n8n'}}`  
   - Connect output of Webhook Trigger to this node.

3. **Add HTTP Request node "Get Current User":**
   - Method: GET  
   - URL: `https://api.calendly.com/users/me`  
   - Authentication: Use Calendly OAuth2 credential (create credential beforehand)  
   - On Error: Continue  
   - Connect from "Set Configuration".

4. **Add a Set node "Extract User":**
   - Assign:  
     - `user_uri` = `{{$json.resource.uri}}`  
     - `user_name` = `{{$json.resource.name}}`  
   - Include other fields from input  
   - Connect from "Get Current User".

5. **Add HTTP Request node "Get Event Types":**
   - Method: GET  
   - URL: `https://api.calendly.com/event_types`  
   - Query Parameters:  
     - `user` = `{{$json.user_uri}}`  
     - `active` = `true`  
   - Authentication: Calendly OAuth2  
   - On Error: Continue  
   - Connect from "Extract User".

6. **Add Set node "Select Event Type":**
   - Assign:  
     - `selected_event_type_uri` = `{{ $('Set Configuration').item.json.requested_event_type || ($json.collection && $json.collection.length > 0 ? $json.collection[0].uri : '') }}`  
     - `selected_event_type_name` = `{{ $json.collection && $json.collection.length > 0 ? $json.collection[0].name : '' }}`  
     - `selected_event_duration` = `{{ $json.collection && $json.collection.length > 0 ? $json.collection[0].duration : 30 }}`  
   - Include other fields  
   - Connect from "Get Event Types".

7. **Add HTTP Request node "Create Single-Use Link":**
   - Method: POST  
   - URL: `https://api.calendly.com/scheduling_links`  
   - Body (JSON):  
     ```json
     {
       "max_event_count": 1,
       "owner": "{{ $json.selected_event_type_uri }}",
       "owner_type": "EventType"
     }
     ```
   - Authentication: Calendly OAuth2  
   - On Error: Continue  
   - Connect from "Select Event Type".

8. **Add Set node "Build Personalized Link":**
   - Assign variables:  
     - `base_booking_url` = `{{$json.resource.booking_url}}`  
     - `personalized_booking_url` = `{{$json.resource.booking_url}}?name={{ encodeURIComponent($('Set Configuration').item.json.recipient_name) }}&email={{ encodeURIComponent($('Set Configuration').item.json.recipient_email) }}&utm_source={{ $('Set Configuration').item.json.utm_source }}`  
     - `recipient_name`, `recipient_email` from "Set Configuration"  
     - `event_type_name`, `event_duration` from "Select Event Type"  
     - `link_created_at` = `{{ new Date().toISOString() }}`  
   - Connect from "Create Single-Use Link".

9. **Add Google Sheets node "Log to Google Sheets":**
   - Operation: Append  
   - Document ID: Your Google Sheets document ID  
   - Sheet Name: Specify sheet or GID  
   - Columns mapped automatically: Recipient Name, Recipient Email, Event Type, Duration (min), Booking URL, Created At, Status (set to "Sent")  
   - Credentials: Google Sheets OAuth2  
   - On Error: Continue  
   - Connect from "Build Personalized Link".

10. **Add Slack node "Notify via Slack":** (Optional)  
    - Channel: your Slack channel ID (e.g., `general`)  
    - Text:  
      ```
      🔗 *New Booking Link Generated*

      👤 *For:* {{ $json.recipient_name }}
      📧 *Email:* {{ $json.recipient_email }}
      📅 *Event:* {{ $json.event_type_name }} ({{ $json.event_duration }} min)

      🔗 <{{ $json.personalized_booking_url }}|Click to Book>
      ```  
    - Credentials: Slack API OAuth token  
    - On Error: Continue  
    - Connect from "Log to Google Sheets".

11. **Add Respond to Webhook node:**
    - Response Code: 200  
    - Headers: Content-Type = application/json  
    - Response Body (JSON):  
      ```json
      {
        "success": true,
        "booking_url": "{{ $('Build Personalized Link').item.json.personalized_booking_url }}",
        "base_url": "{{ $('Build Personalized Link').item.json.base_booking_url }}",
        "recipient": {
          "name": "{{ $('Build Personalized Link').item.json.recipient_name }}",
          "email": "{{ $('Build Personalized Link').item.json.recipient_email }}"
        },
        "event": {
          "name": "{{ $('Build Personalized Link').item.json.event_type_name }}",
          "duration_minutes": {{ $('Build Personalized Link').item.json.event_duration }}
        },
        "created_at": "{{ $('Build Personalized Link').item.json.link_created_at }}",
        "expires": "Single-use or 90 days"
      }
      ```  
    - Connect from "Notify via Slack" or directly from "Log to Google Sheets" if Slack notification is omitted.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Single-use Calendly links expire after one booking or 90 days unused.                                          | Workflow design ensures links are unique and tracked accordingly.                                                |
| Setup Calendly OAuth2 credentials via calendly.com/integrations under API & Webhooks.                          | Required for authentication with Calendly API.                                                                   |
| Google Sheets document must have headers: Name, Email, Link, Event, Created, Duration (min), Status            | For proper logging, spreadsheet must be pre-created and document ID configured in the Google Sheets node.        |
| Slack notification is optional and requires a Slack app with bot token and appropriate scopes configured.     | Slack app setup instructions at https://api.slack.com                                                              |
| Workflow assumes valid recipient emails and names are passed in webhook payload; no complex validation done.  | Input validation can be added upstream if required.                                                              |
| Refer to the sticky notes inside the workflow for detailed setup guides and JSON payload/response formats.    | Visual guidance embedded in the workflow editor.                                                                  |

---

**Disclaimer:**  
The content provided is exclusively derived from an automated n8n workflow. It adheres strictly to current content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly accessible.