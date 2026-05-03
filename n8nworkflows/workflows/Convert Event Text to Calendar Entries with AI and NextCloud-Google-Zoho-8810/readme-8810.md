Convert Event Text to Calendar Entries with AI and NextCloud/Google/Zoho

https://n8nworkflows.xyz/workflows/convert-event-text-to-calendar-entries-with-ai-and-nextcloud-google-zoho-8810


# Convert Event Text to Calendar Entries with AI and NextCloud/Google/Zoho

### 1. Workflow Overview

This n8n workflow automates the conversion of unstructured event text data into calendar entries using AI-powered natural language processing and calendar APIs. It is designed to process event details (e.g., from email bodies or text extracted from images), extract structured event information, and then create events in a calendar system such as NextCloud CalDAV, with optional support for Google Calendar and Zoho Calendar (disabled by default).

The workflow is logically divided into these blocks:

- **1.1 Input Reception**: Receives event information as text via webhook.
- **1.2 Event Parsing with AI**: Uses an AI agent powered by OpenAI to extract structured event details from the input text, applying time zone normalization.
- **1.3 Event Data Structuring**: Parses AI output into a precise JSON schema.
- **1.4 Event Creation in Calendar**: Sends the structured event data to a NextCloud CalDAV calendar to create the event, with optional disabled nodes for Google and Zoho calendars.
- **1.5 Response Handling**: Returns a success or failure message to the webhook source.
- **1.6 Utility Functions**: Includes a helper tool to convert local datetime strings to UTC-formatted strings for calendar compatibility.

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block receives incoming event information as text via an HTTP webhook and prepares it for parsing.

**Nodes Involved:**  
- Inbound Event Info  
- Sticky Note  
- Sticky Note1 (optional image handling note)  

**Node Details:**

- **Inbound Event Info**  
  - Type: Webhook  
  - Role: Entry point that receives POST requests containing event text under `eventInfo` property.  
  - Config: HTTP POST method on path `/make-cal-event-xdt8gh4-rf3827`, set to respond via response node.  
  - Inputs: External webhook HTTP request with JSON body (expects `eventInfo`).  
  - Outputs: Passes eventInfo JSON to the AI parsing node.  
  - Edge cases: No explicit validation here; bad or missing input handled downstream.

- **Sticky Note & Sticky Note1**  
  - Purpose: Documentation and suggestions for input format and optional image processing branch (disabled).  
  - Sticky Note1 mentions a disabled Switch node for image input branching, which is not active but suggests possible workflow extension.

---

#### 2.2 Event Parsing with AI

**Overview:**  
This block invokes an AI agent to analyze unstructured text and extract structured event information, normalizing date-times to UTC.

**Nodes Involved:**  
- Brain (OpenAI language model)  
- Parse Event Info (Langchain Agent)  
- to_UTC (Langchain ToolCode helper)  
- Structured Output (Langchain output parser)  
- Good Parse (If node to check parse success)  

**Node Details:**

- **Parse Event Info**  
  - Type: Langchain Agent (AI)  
  - Role: Extracts event details from input text using an OpenAI chat model and converts local datetimes to UTC using the custom tool `to_UTC`.  
  - Config:  
    - System message defines strict JSON output schema with fields: eventTitle, description, startTime, endTime, location, url.  
    - Special rules for inference if incomplete data, or error output on bad input.  
    - Uses expression to inject eventInfo text and current time zone context.  
    - Calls `to_UTC` tool for datetime conversion.  
  - Inputs: Receives eventInfo text from webhook node.  
  - Outputs: JSON object with parsed event details or error.  
  - Edge cases: AI parsing failure, malformed input, or missing event data triggers error output.

- **to_UTC**  
  - Type: Langchain ToolCode (JavaScript code)  
  - Role: Converts various datetime string formats or Date objects to ICS-compliant UTC datetime strings (e.g., `YYYYMMDDTHHmmssZ`).  
  - Config: Custom JavaScript function handling ISO, ICS, or compact datetime formats robustly.  
  - Inputs: Called internally by AI agent via expressions.  
  - Outputs: UTC-formatted datetime strings.  
  - Edge cases: Invalid date strings throw errors; must be handled by AI prompt logic.

- **Brain**  
  - Type: OpenAI Chat Model Node  
  - Role: Underlying AI model node used by the Langchain agent node (Parse Event Info).  

- **Structured Output**  
  - Type: Langchain Output Parser Structured  
  - Role: Parses AI response into strongly typed JSON object following the schema example provided.  
  - Inputs: AI agent output.  
  - Outputs: Clean JSON for downstream use.

- **Good Parse**  
  - Type: If Node  
  - Role: Checks if parsed output contains no error field to determine if event parsing succeeded.  
  - Inputs: Structured Output node.  
  - Outputs: Routes flow to event creation or failure response.

---

#### 2.3 Event Creation in Calendar

**Overview:**  
This block creates the calendar event in NextCloud CalDAV using the structured event data. Disabled nodes for Google and Zoho calendar creation exist but inactive by default.

**Nodes Involved:**  
- NextCloud Cal Event Creation  
- Create Zoho Event (disabled)  
- Google Calendar (disabled)  
- Sticky Note3  

**Node Details:**

- **NextCloud Cal Event Creation**  
  - Type: HTTP Request  
  - Role: Sends a PUT request to NextCloud CalDAV endpoint to create a new event in iCalendar format.  
  - Config:  
    - URL endpoint set to user’s NextCloud calendar URL (placeholder).  
    - HTTP Method: PUT  
    - Content-Type: text/calendar; charset=utf-8  
    - Body: iCalendar event text dynamically populated with event details: UID, timestamps, summary, description, location, URL.  
    - On error: Continue to catch errors for downstream handling.  
  - Inputs: Structured event JSON from Good Parse node.  
  - Outputs: Success or error response to next block.  
  - Edge cases: Network errors, authentication failures, invalid calendar URLs, malformed event data.

- **Create Zoho Event (API) [Disabled]**  
  - Type: HTTP Request  
  - Role: Placeholder for Zoho Calendar event creation via API with OAuth2 credentials.  
  - Disabled: True  
  - Notes: Requires inserting calendar UID and authenticating with OAuth2.

- **Google Calendar [Disabled]**  
  - Type: Google Calendar Node  
  - Role: Another alternative calendar creation node, disabled.  
  - Config: Requires calendar selection and OAuth2 credentials.

- **Sticky Note3**  
  - Provides instructions for NextCloud Basic Auth credentials and app-specific passwords setup.  
  - Provides useful link for NextCloud user security settings.

---

#### 2.4 Response Handling

**Overview:**  
Sends HTTP responses back to the source system indicating success or failure of event creation.

**Nodes Involved:**  
- Success Response  
- Fail Response  
- Sticky Note4  

**Node Details:**

- **Success Response**  
  - Type: Respond to Webhook  
  - Role: Sends HTTP 201 response with a plain text success message confirming event creation.  
  - Inputs: Triggered on successful calendar event creation.  

- **Fail Response**  
  - Type: Respond to Webhook  
  - Role: Sends HTTP 400 response with a plain text failure message indicating bad event info or failure.  
  - Inputs: Triggered either on parsing failure or event creation error.

- **Sticky Note4**  
  - Explains response purpose and links to a related iCloud Shortcut for seamless integration.

---

#### 2.5 Utility and Documentation

**Overview:**  
Supporting nodes for utility functions and workflow documentation.

**Nodes Involved:**  
- Sticky Note (multiple)  
- Sticky Note5 (Suggestions and best practices)  
- Sticky Note6 (Author credits and branding)  

**Node Details:**

- **Sticky Note5**  
  - Advises on production considerations: error handling, fallback messages, downtime scenarios.  
  - Provides a link to an iOS Shortcut for quick photo-to-calendar event creation.

- **Sticky Note6**  
  - Author information and branding with LinkedIn URL and personal image.

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                         | Input Node(s)              | Output Node(s)                    | Sticky Note                                                  |
|---------------------------|---------------------------------|---------------------------------------|----------------------------|---------------------------------|--------------------------------------------------------------|
| Inbound Event Info         | Webhook                         | Receives event text input              | -                          | Parse Event Info                 | ## START Send a text block of event data (like from email)   |
| Sticky Note               | Sticky Note                    | Documentation                         | -                          | -                               | ## START Send a text block of event data (like from email)   |
| Sticky Note1              | Sticky Note                    | Documentation (optional image input)  | -                          | -                               | ## EXPANSION (Optional image input processing)               |
| Parse Event Info          | Langchain Agent                | AI parsing of event text               | Inbound Event Info          | Good Parse                      | ## PARSE EVENT DETAILS AI agent parses unformatted text       |
| to_UTC                    | Langchain ToolCode             | Converts local datetime to UTC string | Called internally by Parse Event Info | Parse Event Info         | =Call this tool and provide datetime object for UTC string   |
| Brain                     | OpenAI LM Chat                 | AI language model used by agent        | -                          | Parse Event Info                |                                                              |
| Structured Output         | Langchain Output Parser        | Parses AI JSON output                   | Parse Event Info            | Good Parse                      |                                                              |
| Good Parse                | If                            | Checks if parse output is valid        | Structured Output           | NextCloud Cal Event Creation, Fail Response |                                                      |
| NextCloud Cal Event Creation | HTTP Request                  | Creates calendar event in NextCloud    | Good Parse                  | Success Response, Fail Response | ## CREATE EVENT NextCloud CalDAV event creation instructions |
| Create Zoho Event (API)   | HTTP Request (disabled)         | Zoho calendar event creation (disabled) | Good Parse                | -                               | Documentation link to Zoho API                               |
| Google Calendar           | Google Calendar (disabled)      | Google calendar event creation (disabled) | Good Parse                | -                               |                                                              |
| Success Response          | Respond to Webhook             | Sends success HTTP response            | NextCloud Cal Event Creation | -                             | ## RESPOND success message with shortcut link                |
| Fail Response             | Respond to Webhook             | Sends failure HTTP response            | Good Parse, NextCloud Cal Event Creation | -                      | ## RESPOND failure message                                    |
| Sticky Note2              | Sticky Note                    | Documentation                         | -                          | -                               | ## PARSE EVENT DETAILS AI agent parses unformatted text       |
| Sticky Note3              | Sticky Note                    | NextCloud calendar creation instructions | -                        | -                               | ## CREATE EVENT NextCloud CalDAV event creation instructions |
| Sticky Note4              | Sticky Note                    | Response handling instructions         | -                          | -                               | ## RESPOND success message with shortcut link                |
| Sticky Note5              | Sticky Note                    | Suggestions for production usage       | -                          | -                               | Production error handling, iOS Shortcut link                  |
| Sticky Note6              | Sticky Note                    | Author branding and credits            | -                          | -                               | Eric Knaus LinkedIn and branding                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node "Inbound Event Info":**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `make-cal-event-xdt8gh4-rf3827`  
   - Response Mode: Respond with node  
   - Expect JSON body with property `eventInfo` containing event text.

2. **Create Langchain ToolCode Node "to_UTC":**  
   - Type: Langchain ToolCode  
   - Name: `to_UTC`  
   - JavaScript code: Function to convert input datetime (Date object or string in various formats) to UTC ICS datetime string (`YYYYMMDDTHHmmssZ`).  
   - Description: Call this tool with local datetime to get UTC string.  
   - Version: 1.1

3. **Create OpenAI Chat Model Node "Brain":**  
   - Type: Langchain LM Chat OpenAI  
   - Leave default settings or configure API credentials as needed.

4. **Create Langchain Agent Node "Parse Event Info":**  
   - Type: Langchain Agent  
   - Text parameter: Use expression to inject local time context and input text `{{ $json.body.eventInfo }}`.  
   - System Message:  
     - Instruct to extract structured JSON event details with fields: eventTitle, description, startTime, endTime, location, url.  
     - Include special rules to infer missing times or return error JSON for bad input.  
     - Use `to_UTC` tool for datetime conversions.  
   - Link to "Brain" node as language model.  
   - Link to "to_UTC" node as tool.

5. **Create Langchain Output Parser Node "Structured Output":**  
   - Type: Langchain Output Parser Structured  
   - Provide example JSON schema to enforce output format matching expected event fields.

6. **Create If Node "Good Parse":**  
   - Condition: Check if output JSON does NOT contain an `error` key.  
   - True branch: Proceed to event creation.  
   - False branch: Trigger failure response.

7. **Create HTTP Request Node "NextCloud Cal Event Creation":**  
   - URL: Your NextCloud CalDAV event creation endpoint (e.g., `https://your.nextcloudurl.com/remote.php/dav/calendars/YOUR_USER/personal/newEvent.ics`)  
   - Method: PUT  
   - Headers: Content-Type `text/calendar; charset=utf-8`  
   - Body (Raw, text/calendar format):  
     ```
     BEGIN:VCALENDAR
     VERSION:2.0
     PRODID:-//n8n//CalDAV Connector//EN
     BEGIN:VEVENT
     UID:123456789@example.com
     DTSTAMP:{{ $now.setZone($now.zone.zoneName).toUTC().toFormat("yyyyMMdd'T'HHmmss'Z'") }}
     DTSTART:{{ $json.output.startTime }}
     DTEND:{{ $json.output.endTime }}
     SUMMARY:{{ $json.output.eventTitle }}
     DESCRIPTION:{{ $json.output.description }}
     LOCATION:{{ $json.output.location }}
     URL:{{ $json.output.url }}
     END:VEVENT
     END:VCALENDAR
     ```
   - On Error: Continue on error to allow failure handling downstream.

8. **Create Respond to Webhook Node "Success Response":**  
   - Response Code: 201  
   - Response Body (Text): "Your calendar event was successfully created."  
   - Triggered on successful event creation.

9. **Create Respond to Webhook Node "Fail Response":**  
   - Response Code: 400  
   - Response Body (Text): "There was a problem with the event info. Try again."  
   - Triggered on parsing failure or event creation error.

10. **Connect Nodes:**  
    - Webhook → Parse Event Info  
    - Parse Event Info → Structured Output → Good Parse  
    - Good Parse (true) → NextCloud Cal Event Creation → Success Response  
    - Good Parse (false) → Fail Response  
    - NextCloud Cal Event Creation (error output) → Fail Response

11. **Credentials:**  
    - OpenAI API credentials configured for "Brain" node.  
    - NextCloud Basic Auth credentials configured for HTTP Request node.  
    - (Optional) OAuth2 credentials for Zoho and Google calendar nodes if enabling.

12. **Optional:**  
    - Add Sticky Notes for documentation and usage guidance at relevant points.  
    - Enable and configure the Switch node and image processing if handling image inputs.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                 | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| This workflow plugs in seamlessly with an iCloud Shortcut allowing iPhone users to snap a photo and add calendar events via Siri.                                                                            | https://www.icloud.com/shortcuts/8a107ea08ec4471d877b019520a4802c                                   |
| For NextCloud, create an app-specific password via your user security settings to enable Basic Auth calendar API access.                                                                                    | NextCloud user security settings: [your.NC.url/index.php/settings/user/security]                   |
| Suggestions for production use: handle errors such as malformed input, NextCloud downtime, and instruct the AI to output specific fail messages for better UX.                                               | See Sticky Note5 content                                                                           |
| Author: Eric Knaus — MarketingGuy.ai. LinkedIn: https://linkedin.com/in/ericknaus. Personal branding image linked in notes.                                                                                   | https://linkedin.com/in/ericknaus                                                                  |
| Zoho Calendar API documentation is available but this node is currently disabled. OAuth2 credentials required for usage.                                                                                     | https://www.zoho.com/calendar/help/api/post-create-event.html                                     |

---

This completes the comprehensive analysis and documentation of the workflow "Convert Event Text to Calendar Entries with AI and NextCloud/Google/Zoho". It provides all details necessary for understanding, modification, troubleshooting, and manual recreation.