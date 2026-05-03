Create Google Calendar events from labeled Gmail emails with Google Gemini AI

https://n8nworkflows.xyz/workflows/create-google-calendar-events-from-labeled-gmail-emails-with-google-gemini-ai-7340


# Create Google Calendar events from labeled Gmail emails with Google Gemini AI

### 1. Workflow Overview

This workflow automates the creation of Google Calendar events from Gmail emails labeled with a specific tag (e.g., "Scheduled"), leveraging Google Gemini AI to parse unstructured email content into structured event data. It targets users who receive event-related information via email and want to streamline adding these events to their calendar without manual entry.

The workflow is logically divided into four blocks:

- **1.1 Email Trigger:** Watches for new emails with a designated Gmail label to initiate the workflow.
- **1.2 AI Parsing of Event Details:** Sends the email content to Google Gemini AI to extract structured event details (title, time, location, description).
- **1.3 Calendar Event Creation:** Creates a new event in Google Calendar using the parsed data.
- **1.4 Confirmation Email:** Sends a confirmation email summarizing the created event and including a link to edit it in Google Calendar, as well as the original email content for reference.

---

### 2. Block-by-Block Analysis

#### 1.1 Email Trigger

- **Overview:**  
  Initiates the workflow when a new email in Gmail receives a specific user-defined label. This allows selective triggering based on email categorization.

- **Nodes Involved:**  
  - Gmail Trigger

- **Node Details:**

  - **Gmail Trigger**  
    - *Type & Role:* Event trigger node that listens for incoming Gmail emails with specified labels.  
    - *Configuration:*  
      - `Label Ids`: User selects the Gmail label(s) to listen for (e.g., "Scheduled").  
      - Polling frequency: Every minute.  
      - `Simple`: false (to get detailed email data).  
    - *Expressions/Variables:* None configured beyond label filter.  
    - *Connections:* Outputs to "Parse Event with AI".  
    - *Version-specific:* Uses version 1.2 of Gmail Trigger node.  
    - *Failure cases:* Authentication errors (invalid Gmail credentials), polling delays, label ID misconfiguration leading to no triggers.  
    - *Sub-workflow:* None.

---

#### 1.2 AI Parsing of Event Details

- **Overview:**  
  Transforms unstructured email text into a structured JSON format representing event details, using Google Gemini AI with a Langchain agent setup and a structured output parser.

- **Nodes Involved:**  
  - Parse Event with AI  
  - Google Gemini Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **Parse Event with AI**  
    - *Type & Role:* Langchain agent node configured to generate structured JSON from email text.  
    - *Configuration:*  
      - A detailed prompt defines the expected JSON schema including summary, start and end datetimes (with timezone), location, and description.  
      - The prompt instructs the AI to infer missing date/time if not explicit and use 24-hour format.  
      - Input text comes from the Gmail Trigger node’s email content (`$json["text"]`).  
      - Output is parsed as structured JSON.  
    - *Expressions/Variables:* Uses expression to embed email text into prompt.  
    - *Connections:* Main output to "Create Google Calendar Event".  
      - AI language model input connected to "Google Gemini Chat Model".  
      - AI output parser input connected to "Structured Output Parser".  
    - *Version-specific:* Uses version 2.1, supporting advanced prompt and output parser integration.  
    - *Failure cases:*  
      - AI misinterpretation or incomplete parsing.  
      - Timeout or API errors communicating with Gemini.  
      - Schema mismatch causing parser failure.  
    - *Sub-workflow:* None.

  - **Google Gemini Chat Model**  
    - *Type & Role:* Language model node that calls Google Gemini API for chat completions.  
    - *Configuration:* Default options (credentials must be set up).  
    - *Connections:* Outputs to "Parse Event with AI" node’s AI language model input.  
    - *Failure cases:* Authentication failures, API quota limits, network issues.  
    - *Sub-workflow:* None.

  - **Structured Output Parser**  
    - *Type & Role:* Parses the AI response to enforce structured JSON output based on provided schema example.  
    - *Configuration:* Uses a JSON schema example matching the expected event object structure.  
    - *Connections:* Output feeds back into "Parse Event with AI" for main output.  
    - *Failure cases:* Parsing errors if AI output deviates from schema.  
    - *Sub-workflow:* None.

---

#### 1.3 Calendar Event Creation

- **Overview:**  
  Creates a new event in the user's primary Google Calendar using the structured event information extracted by AI.

- **Nodes Involved:**  
  - Create Google Calendar Event

- **Node Details:**

  - **Create Google Calendar Event**  
    - *Type & Role:* Google Calendar node that inserts a new calendar event.  
    - *Configuration:*  
      - Calendar: Primary calendar selected by ID.  
      - Event fields populated with AI output data:  
        - `summary` from `$json.output.summary`  
        - `start.dateTime` and `end.dateTime` from AI output (ISO8601 with timezone)  
        - `location` and `description` from AI output  
    - *Expressions/Variables:* Multiple expressions pull data from AI JSON output.  
    - *Connections:* Outputs to "Send Confirmation Email".  
    - *Failure cases:*  
      - Authentication errors with Google Calendar credentials.  
      - Invalid or missing date/time formats causing API rejection.  
      - Quota limits or API downtime.  
    - *Sub-workflow:* None.

---

#### 1.4 Confirmation Email

- **Overview:**  
  Sends a detailed confirmation email summarizing the newly created calendar event, including a direct link to edit the event and the original email content for reference.

- **Nodes Involved:**  
  - Send Confirmation Email

- **Node Details:**

  - **Send Confirmation Email**  
    - *Type & Role:* Gmail node configured to send an email notification.  
    - *Configuration:*  
      - `Send To`: User must replace placeholder `YOUR_EMAIL_ADDRESS` with their destination email address.  
      - Email subject dynamically includes event title (`Event Created in Google Calendar: {{ $json.summary }}`).  
      - Email body is HTML formatted, includes:  
        - Event title, times (start/end formatted to locale string), location, description (with new lines converted to `<br>`).  
        - Link to edit the event (`$json.htmlLink`).  
        - Original email content displayed in a `<pre>` block for clarity.  
      - Attribution disabled, no attachments.  
    - *Expressions/Variables:* Multiple template expressions referencing event data.  
    - *Connections:* None (end of workflow).  
    - *Failure cases:*  
      - Gmail authentication failure.  
      - Invalid recipient email address or sending limits.  
    - *Sub-workflow:* None.

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                   | Input Node(s)          | Output Node(s)           | Sticky Note                                                                                                                               |
|---------------------------|----------------------------------|---------------------------------|-----------------------|--------------------------|-------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow Description      | Sticky Note                      | Documentation / Overview         |                       |                          | ## Create Google Calendar events from labeled Gmail emails using AI... (full description of purpose, audience, setup, customization)     |
| Gmail Trigger             | Gmail Trigger                    | Email trigger on labeled emails |                       | Parse Event with AI       | ## 1. Email Trigger                                                                                                                        |
| Parse Event with AI       | Langchain Agent                  | Parse email text to event JSON  | Gmail Trigger          | Create Google Calendar Event | ## 2. Parse Event with AI                                                                                                                  |
| Google Gemini Chat Model  | Langchain LM Chat (Google Gemini) | Provides AI model response      |                       | Parse Event with AI (ai_languageModel) | ## 2. Parse Event with AI                                                                                                                  |
| Structured Output Parser  | Langchain Output Parser Structured | Parses AI output into JSON      |                       | Parse Event with AI (ai_outputParser) | ## 2. Parse Event with AI                                                                                                                  |
| Create Google Calendar Event | Google Calendar                | Creates calendar event          | Parse Event with AI    | Send Confirmation Email   | ## 3. Create Calendar Event                                                                                                                |
| Send Confirmation Email   | Gmail                           | Sends event confirmation email  | Create Google Calendar Event |                          | ## 4. Send Confirmation Email                                                                                                              |
| Sticky Note               | Sticky Note                     | Block label 1.1                 |                       |                          | ## 1. Email Trigger                                                                                                                        |
| Sticky Note1              | Sticky Note                     | Block label 1.2                 |                       |                          | ## 2. Parse Event with AI                                                                                                                  |
| Sticky Note2              | Sticky Note                     | Block label 1.3                 |                       |                          | ## 3. Create Calendar Event                                                                                                                |
| Sticky Note3              | Sticky Note                     | Block label 1.4                 |                       |                          | ## 4. Send Confirmation Email                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger node**  
   - Type: Gmail Trigger  
   - Configuration:  
     - Select Gmail credentials.  
     - Set `Label Ids` to the Gmail label that should trigger the workflow (e.g., "Scheduled").  
     - Set polling interval to every minute.  
     - Set `Simple` to false to get detailed email content.  
   - Connect output to next node.

2. **Create Google Gemini Chat Model node**  
   - Type: Langchain LM Chat Google Gemini  
   - Configure with Google Gemini credentials.  
   - Leave options default.  
   - This node will be linked as the AI language model input for the Langchain agent node.

3. **Create Structured Output Parser node**  
   - Type: Langchain Output Parser Structured  
   - Configure with JSON schema example:  
     ```json
     {
       "summary": "Event title (e.g., Meeting, Appointment, Task)",
       "start": { "dateTime": "YYYY-MM-DDTHH:MM:SS+09:00" },
       "end": { "dateTime": "YYYY-MM-DDTHH:MM:SS+09:00" },
       "location": "Location (e.g., URL for online meetings, Meeting Room A, Cafe)",
       "description": "Detailed description of the event (e.g., the original email content)"
     }
     ```
   - This node will be linked as the AI output parser input for the Langchain agent node.

4. **Create Parse Event with AI node**  
   - Type: Langchain Agent  
   - Set prompt type: Define  
   - Insert prompt text (adapted as needed):  
     ```
     Extract the Google Calendar event information from the following email content into JSON format.

     JSON format:
     {
       "summary": "Event title (e.g., Meeting, Appointment, Task)",
       "start": { "dateTime": "YYYY-MM-DDTHH:MM:SS+09:00" },
       "end": { "dateTime": "YYYY-MM-DDTHH:MM:SS+09:00" },
       "location": "Location (e.g., URL for online meetings, Meeting Room A, Cafe)",
       "description": "Detailed description of the event (should contain the original email content)"
     }

     If the date and time are not explicit, infer them based on the current time or set the most likely period. Use 24-hour format for time. Please ensure the timezone is correct for your location (e.g., change 'JST (UTC+9)' if needed). The location is optional.

     Email content:
     {{$json["text"]}}
     ```
   - Connect AI language model input to "Google Gemini Chat Model" node.  
   - Connect AI output parser input to "Structured Output Parser" node.  
   - Connect main output to next node.

5. **Connect Gmail Trigger main output to Parse Event with AI main input.**

6. **Create Google Calendar Event node**  
   - Type: Google Calendar  
   - Select Google Calendar credentials.  
   - Set calendar to "primary" by ID.  
   - Configure event fields with expressions:  
     - `summary`: `={{ $json.output.summary }}`  
     - `start`: `={{ $json.output.start.dateTime }}`  
     - `end`: `={{ $json.output.end.dateTime }}`  
     - `location`: `={{ $json.output.location }}` (optional)  
     - `description`: `={{ $json.output.description }}`  
   - Connect input from "Parse Event with AI" main output.  
   - Connect output to next node.

7. **Create Send Confirmation Email node**  
   - Type: Gmail  
   - Select Gmail credentials.  
   - Configure:  
     - `Send To`: replace `YOUR_EMAIL_ADDRESS` with your actual email address.  
     - `Subject`: `=Event Created in Google Calendar: {{ $json.summary }}`  
     - `Message`: Use HTML content with embedded expressions:  
       ```html
       <html><body>
           <p>An event has been created in your Google Calendar with the following details.</p>
           <hr>
           <h2>Event Summary:</h2>
           <p><strong>Title:</strong> {{ $json.summary }}</p>
           <p><strong>Time:</strong> {{ new Date($json.start.dateTime).toLocaleString() }} - {{ new Date($json.end.dateTime).toLocaleString() }}</p>
           <p><strong>Location:</strong> {{ $json.location }}</p>
           <p><strong>Description:</strong><br>{{ $json.description.replace(/\n/g, '<br>') }}</p>
           <p><a href="{{ $json.htmlLink }}">Edit in Google Calendar</a></p>
           <hr>
           <h2>Original Email Content:</h2>
           <pre>{{ $json.description }}</pre>
       </body></html>
       ```  
     - Disable attribution.  
     - No attachments.  
   - Connect input from "Create Google Calendar Event" output.

8. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                               | Context or Link                                                     |
|------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| This workflow requires Google account credentials configured for Gmail, Google Calendar, and Google Gemini API. | Credentials setup instructions in n8n documentation.              |
| The AI prompt in the "Parse Event with AI" node can be customized for different timezones or recurring events. | Adjust prompt text inside the node.                                |
| Use a Gmail label such as "Scheduled" to trigger event creation only for relevant emails.                 | Gmail label configuration in Gmail settings.                      |
| Confirmation email content is fully customizable with HTML and expressions.                               | Modify "Send Confirmation Email" node parameters accordingly.     |
| For best results, ensure Google Gemini API quota and authentication are valid to avoid failures.          | Google Gemini API documentation and n8n credential management.    |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.