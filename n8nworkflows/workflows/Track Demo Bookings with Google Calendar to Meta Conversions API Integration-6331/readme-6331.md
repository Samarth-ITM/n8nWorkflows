Track Demo Bookings with Google Calendar to Meta Conversions API Integration

https://n8nworkflows.xyz/workflows/track-demo-bookings-with-google-calendar-to-meta-conversions-api-integration-6331


# Track Demo Bookings with Google Calendar to Meta Conversions API Integration

### 1. Workflow Overview

This workflow automates tracking of demo bookings created in Google Calendar by sending corresponding conversion events to Meta’s (Facebook) Conversion API. It is designed for businesses that schedule demos via Google Calendar and want to track these leads in Meta’s advertising ecosystem to optimize ad targeting and performance measurement.

**Target Use Cases:**
- Automatically capturing demo scheduling events from Google Calendar.
- Filtering only relevant demo events by event name.
- Formatting attendee information securely.
- Sending “Schedule” conversion events to Meta’s Conversion API for ad tracking.

**Logical Blocks:**

- **1.1 Event Trigger and Configuration:** Captures new events from Google Calendar and sets configuration parameters.
- **1.2 Event Filtering:** Filters incoming calendar events to process only those matching the demo event name.
- **1.3 Data Formatting and Hashing:** Hashes attendee email and formats event data for Meta API.
- **1.4 Meta Conversion API Request:** Sends the conversion event to Meta’s API.
- **1.5 No-Operation Path:** Handles non-matching events by doing nothing.
- **1.6 Documentation and Notes:** Contains sticky notes explaining workflow usage and configurations.

---

### 2. Block-by-Block Analysis

#### 2.1 Event Trigger and Configuration

- **Overview:**  
  This block listens for new events created in a specified Google Calendar and initializes configuration parameters needed for the Meta API call.

- **Nodes Involved:**  
  - Google Calendar Trigger  
  - Config

- **Node Details:**

  - **Google Calendar Trigger**  
    - *Type & Role:* Trigger node that listens to Google Calendar events.  
    - *Configuration:* Polls every minute for newly created events (`triggerOn: eventCreated`). Targets a specific calendar identified by the user’s email.  
    - *Inputs/Outputs:* No input; outputs event JSON when a new calendar event is created.  
    - *Credentials:* Uses Google OAuth2 credentials configured via Google Console API.  
    - *Potential Failures:* Auth token expiration, API quota limits, no calendar access permissions.  
    - *Notes:* Must ensure the calendarId is correctly set to the user’s calendar.

  - **Config**  
    - *Type & Role:* Set node to define Meta API configuration parameters.  
    - *Configuration:*  
      - `FB_PIXEL_ID`: Facebook Pixel identifier (string).  
      - `FB_ACCESS_TOKEN`: Access token for Meta API authentication.  
      - `FB_API_VERSION`: API version, default “v22.0”.  
      - `source_url`: Landing page URL.  
      - `event_name_filter`: String used to filter demo event names.  
    - *Inputs:* Receives event data from Google Calendar Trigger.  
    - *Outputs:* Passes event data forward with added config values.  
    - *Edge Cases:* Misconfigured or missing parameters will cause failures downstream.  
    - *Notes:* Values must be replaced with actual credentials and URLs before activation.

---

#### 2.2 Event Filtering

- **Overview:**  
  Filters incoming calendar events to process only those whose summary (event name) contains the configured demo event name filter.

- **Nodes Involved:**  
  - If  
  - No Operation, do nothing

- **Node Details:**

  - **If**  
    - *Type & Role:* Conditional node that evaluates event summary.  
    - *Configuration:* Uses an expression to check if the event’s summary string contains the `event_name_filter` string (case-sensitive, strict validation).  
    - *Input:* Receives event data + config from the Config node.  
    - *Output:*  
      - True branch: Event matches filter, proceed to data processing.  
      - False branch: Event does not match filter, lead to no-op node.  
    - *Edge Cases:* If event summary is missing or empty, condition may fail silently. Improper filter string can cause no matches.  
    - *Version:* Uses condition version 2.2 for advanced string operations.

  - **No Operation, do nothing**  
    - *Type & Role:* No-op node to explicitly handle filtered out events by doing nothing.  
    - *Input:* False branch from If node.  
    - *Output:* None (end of flow for non-matching events).  
    - *Purpose:* Prevents further processing and API calls for irrelevant events.

---

#### 2.3 Data Formatting and Hashing

- **Overview:**  
  Processes the filtered event data by hashing the attendee’s email and assembling the required parameters for the Meta Conversion API payload.

- **Nodes Involved:**  
  - Crypto  
  - Params

- **Node Details:**

  - **Crypto**  
    - *Type & Role:* Cryptographic node to hash sensitive user data.  
    - *Configuration:*  
      - Type: SHA256 (secure hash algorithm).  
      - Value: The email address of the second attendee in the event (`attendees[1].email`).  
      - Stores result in `hashedEmail`.  
    - *Input:* Event data from the If node’s true branch.  
    - *Output:* Appends hashed email to JSON data.  
    - *Edge Cases:* If attendee email is missing or array index out of bounds, will error or produce invalid hash.  
    - *Version:* Version 1.

  - **Params**  
    - *Type & Role:* Set node to prepare the final parameters for API request.  
    - *Configuration:* Assigns the following parameters:  
      - `em`: hashedEmail from Crypto node.  
      - `client_ip_address`: from JSON input `clientIp`, defaults to `0.0.0.0` if missing.  
      - `client_user_agent`: from JSON input `userAgent`, defaults to `n8n/MetaCAPI`.  
      - `event_time`: current Unix timestamp in seconds (calculated dynamically).  
    - *Input:* Output from Crypto node.  
    - *Output:* Passes formatted parameters forward to HTTP request node.  
    - *Potential Issues:* Missing or malformed input data may cause incomplete parameter sets.

---

#### 2.4 Meta Conversion API Request

- **Overview:**  
  Sends a POST request to the Facebook Graph API to register the “Schedule” event with all required user and event data.

- **Nodes Involved:**  
  - HTTP Request - Meta Conversion API

- **Node Details:**

  - **HTTP Request - Meta Conversion API**  
    - *Type & Role:* HTTP Request node to perform the API call.  
    - *Configuration:*  
      - URL dynamically constructed from Config node values:  
        `https://graph.facebook.com/{FB_API_VERSION}/{FB_PIXEL_ID}/events`  
      - Method: POST  
      - Content-Type: multipart/form-data (header explicitly set to application/json, which may need consistency check)  
      - Body Parameters:  
        - `data`: JSON array representing the event, including:  
          - `event_name`: “Schedule”  
          - `event_time`: from Params node  
          - `action_source`: “website”  
          - `event_source_url`: from Config  
          - `user_data`: includes hashed email, client IP, user agent  
          - `attribution_data`: attribution share set to 1.0  
          - `original_event_data`: replicates event name and time  
        - `access_token`: from Config node  
    - *Input:* Parameters from Params node.  
    - *Output:* API response data.  
    - *Edge Cases & Failure Modes:*  
      - Network failures or timeouts.  
      - Invalid or expired access token.  
      - Incorrect pixel ID or API version.  
      - Malformed payload causing API errors.  
    - *Version:* Version 4.2 HTTP node.

---

#### 2.5 Documentation and Notes

- **Overview:**  
  Provides inline documentation and configuration guidance through sticky notes.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2

- **Node Details:**

  - **Sticky Note**  
    - *Role:* Introduction and high-level workflow explanation.  
    - *Content Highlights:*  
      - Workflow purpose and sequence description.  
      - Required accesses and credential links.  
      - Author contact information.

  - **Sticky Note1**  
    - *Role:* Details on configuration parameters users must set before use.  
    - *Content:* Explains Facebook pixel ID, access token, API version, source URL, and event name filter.

  - **Sticky Note2**  
    - *Role:* Provides additional resource for customizing API payloads.  
    - *Content:* Link to Meta Payload Helper for building additional properties:  
      https://developers.facebook.com/docs/marketing-api/conversions-api/payload-helper/

---

### 3. Summary Table

| Node Name                     | Node Type               | Functional Role                      | Input Node(s)                 | Output Node(s)                    | Sticky Note                                                                                                  |
|-------------------------------|-------------------------|------------------------------------|------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| Google Calendar Trigger        | Google Calendar Trigger | Trigger on new calendar event      | N/A                          | Config                          |                                                                                                              |
| Config                        | Set                     | Defines Meta API and filter params | Google Calendar Trigger       | If                             | **Here you'll configure:** - Facebook Pixel ID - Facebook Access Token - Facebook API Version - Source URL - Event Name Filter |
| If                            | If                      | Filters events by demo event name  | Config                       | Crypto (true branch), No Operation (false branch) |                                                                                                              |
| No Operation, do nothing       | NoOp                    | Ends flow for irrelevant events    | If (false branch)             | N/A                             |                                                                                                              |
| Crypto                       | Crypto                  | Hashes attendee email              | If (true branch)              | Params                         |                                                                                                              |
| Params                       | Set                     | Prepares parameters for API call   | Crypto                       | HTTP Request - Meta Conversion API |                                                                                                              |
| HTTP Request - Meta Conversion API | HTTP Request            | Sends data to Meta Conversion API  | Params                       | N/A                             | **Sending additional properties** Use https://developers.facebook.com/docs/marketing-api/conversions-api/payload-helper/ |
| Sticky Note                  | Sticky Note             | Workflow intro and instructions    | N/A                          | N/A                             | # Welcome to Meta Conversion API Workflow! If you want to send a conversion event to Meta every time a new lead schedules a demo meeting with you, this workflow is for you! 😉 |
| Sticky Note1                 | Sticky Note             | Configuration guidance             | N/A                          | N/A                             | **Here you'll configure:** Facebook Pixel ID, Access Token, API version, Source URL, Event Name Filter         |
| Sticky Note2                 | Sticky Note             | Additional resource link           | N/A                          | N/A                             | ## Sending additional properties Use this link to help build your payload: https://developers.facebook.com/docs/marketing-api/conversions-api/payload-helper/ |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Google Calendar Trigger Node:**  
   - Type: Google Calendar Trigger  
   - Set to poll every minute (`pollTimes` with mode `everyMinute`).  
   - Configure to trigger on event creation (`triggerOn: eventCreated`).  
   - Set `calendarId` to your Google calendar email.  
   - Authenticate with Google OAuth2 credentials configured in n8n.

2. **Create a Set Node Named “Config”:**  
   - Add parameters:  
     - `FB_PIXEL_ID`: Your Facebook Pixel ID (string).  
     - `FB_ACCESS_TOKEN`: Your Meta API access token (string).  
     - `FB_API_VERSION`: “v22.0” (default, string).  
     - `source_url`: Your landing page URL (string).  
     - `event_name_filter`: The exact or partial demo event name to filter on (string).  
   - Connect “Google Calendar Trigger” output to this node input.

3. **Create an If Node:**  
   - Set condition: Check if `{{$json.summary}}` contains `{{$json.event_name_filter}}`.  
   - Use condition version 2.2 with strict validation and case-sensitive string contains operation.  
   - Connect “Config” node output to this node input.

4. **Create a No Operation Node:**  
   - This node handles the false branch of the If node (events not matching filter).  
   - Connect “If” node’s false output to this node.

5. **Create a Crypto Node:**  
   - Type: Crypto  
   - Operation: SHA256 hash  
   - Value: `{{$json.attendees[1].email}}` (second attendee’s email).  
   - Store result in property `hashedEmail`.  
   - Connect “If” node’s true output to this node.

6. **Create a Set Node Named “Params”:**  
   - Define the following parameters:  
     - `em`: `{{$json.hashedEmail}}` (hashed email).  
     - `client_ip_address`: `{{$json.clientIp || '0.0.0.0'}}` (fallback to default IP).  
     - `client_user_agent`: `{{$json.userAgent || 'n8n/MetaCAPI'}}` (fallback user agent).  
     - `event_time`: Current Unix timestamp in seconds: `{{Math.floor(Date.now() / 1000)}}`.  
   - Connect output of “Crypto” node to this node.

7. **Create an HTTP Request Node Named “HTTP Request - Meta Conversion API”:**  
   - Method: POST  
   - URL: `https://graph.facebook.com/{{$('Config').item.json.FB_API_VERSION}}/{{$('Config').item.json.FB_PIXEL_ID}}/events`  
   - Headers: `Content-Type: application/json`  
   - Body Type: Send as JSON (verify if multipart-form-data is needed or JSON)  
   - Body Parameters:  
     - `data`: JSON array with event details, including event_name “Schedule”, event_time from Params, action_source “website”, event_source_url from Config, user_data with hashed email, client IP, user agent, attribution_data, and original_event_data.  
     - `access_token`: From Config node.  
   - Connect output of “Params” node to this node.

8. **Add Sticky Notes for Documentation:**  
   - Create sticky notes with the workflow overview, configuration instructions, and links to Meta documentation (optional but recommended for clarity).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                          | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Google OAuth2 setup is required for Google Calendar Trigger node. See [n8n Google OAuth2 Docs](https://docs.n8n.io/integrations/builtin/credentials/google/oauth-single-service/) | Credential setup for Google API access                                                             |
| Meta Conversion API documentation and setup guide.                                                                                                                  | [Meta Conversion API Get Started](https://developers.facebook.com/docs/marketing-api/conversions-api/get-started) |
| Meta Payload Helper tool to customize conversion event payloads.                                                                                                    | [Meta Payload Helper](https://developers.facebook.com/docs/marketing-api/conversions-api/payload-helper/) |
| Author contact for questions: Marcelo Miranda LinkedIn https://www.linkedin.com/in/marceloamiranda                                                                   | Workflow author and support contact                                                                |

---

This documentation fully details the workflow structure, logic, configuration, and reproduction instructions, enabling effective understanding, modification, and error anticipation.