Generate AI Meeting Notes from Zoom with GPT-4, Google Docs & Slack

https://n8nworkflows.xyz/workflows/generate-ai-meeting-notes-from-zoom-with-gpt-4--google-docs---slack-8057


# Generate AI Meeting Notes from Zoom with GPT-4, Google Docs & Slack

### 1. Workflow Overview

This n8n workflow automates the generation of AI-powered meeting notes from Zoom meetings using GPT-4, and subsequently shares the notes via Google Docs and Slack. It is designed to trigger upon the end of a Zoom meeting, normalize and extract relevant meeting data, generate a summarized transcript or notes through OpenAI’s GPT-4, save the results to Google Docs, and notify a Slack channel with a link to the document.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**: Receives Zoom meeting ended event data via webhook.
- **1.2 Data Normalization**: Extracts and normalizes meeting metadata from the Zoom payload.
- **1.3 AI Processing**: Uses OpenAI GPT-4 to generate summarized meeting notes.
- **1.4 Document Storage**: Saves the AI-generated notes to Google Docs.
- **1.5 Notification Dispatch**: Posts a notification to Slack with meeting details and a link to the Google Doc.
- **1.6 Setup Guidance**: Provides setup instructions as a sticky note for easy configuration reference.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block listens for Zoom’s "meeting ended" webhook and triggers the workflow when a meeting concludes.

- **Nodes Involved:**  
  - Zoom Meeting Webhook

- **Node Details:**  
  - **Zoom Meeting Webhook**  
    - Type: Webhook (HTTP POST listener)  
    - Configuration: Listens on path `/zoom-meeting-ended` for POST requests with Zoom meeting ended event payloads.  
    - Expressions/Variables: None (receives raw Zoom payload)  
    - Input Connections: None (entry point)  
    - Output Connections: Sends data to Normalize Data node  
    - Version: v1  
    - Failure Modes:  
      - Missing or malformed payloads  
      - Incorrect webhook URL configuration in Zoom app  
      - Network or authentication errors on Zoom side  
    - Notes: This node must be publicly accessible and correctly registered as the Zoom event webhook.

#### 1.2 Data Normalization

- **Overview:**  
  This block extracts relevant meeting information from the raw Zoom webhook payload and normalizes it into a simplified JSON structure for downstream processing.

- **Nodes Involved:**  
  - Normalize Data

- **Node Details:**  
  - **Normalize Data**  
    - Type: Code (JavaScript)  
    - Configuration: Custom JS code extracts fields like meeting ID, topic, host email, start/end times, duration, participant count, and transcript if available. Adds a timestamp of processing.  
    - Key Expressions: Accesses `$input.first().json.payload.object` fields for data extraction.  
    - Inputs: Zoom Meeting Webhook output  
    - Outputs: Cleaned JSON ready for AI processing  
    - Version: v2 (uses updated code node version)  
    - Failure Modes:  
      - Missing expected fields in Zoom payload  
      - Transcript may be empty or missing  
      - JavaScript runtime errors if payload structure changes  
    - Notes: Ensures consistent data format for AI node.

#### 1.3 AI Processing

- **Overview:**  
  This block sends the normalized meeting data to OpenAI's GPT-4 to generate concise, human-readable meeting notes.

- **Nodes Involved:**  
  - Generate AI Notes

- **Node Details:**  
  - **Generate AI Notes**  
    - Type: OpenAI (Chat Completion)  
    - Configuration: Uses chat resource with GPT-4 model (implied by setup instructions), no additional request options specified.  
    - Key Expressions: Uses normalized data JSON as prompt content (assumed via input chaining)  
    - Inputs: Output from Normalize Data  
    - Outputs: AI-generated meeting notes text  
    - Version: v1  
    - Failure Modes:  
      - API key missing or invalid  
      - Rate limiting or quota exceeded  
      - Network timeouts or API errors  
      - GPT-4 model availability  
    - Notes: Requires OpenAI API credentials configured in n8n.

#### 1.4 Document Storage

- **Overview:**  
  Saves the AI-generated notes into a new Google Document for easy access and sharing.

- **Nodes Involved:**  
  - Save to Google Docs

- **Node Details:**  
  - **Save to Google Docs**  
    - Type: Google Docs node  
    - Configuration: Creates a new document titled using meeting topic and timestamp from normalized data, content is the AI notes text.  
    - Key Expressions: Title template `Meeting Notes - {{ $json.topic }} - {{ $json.timestamp }}`  
    - Inputs: Output from Generate AI Notes  
    - Outputs: Metadata including `webViewLink` to the created document  
    - Version: v2  
    - Failure Modes:  
      - Google OAuth token missing or expired  
      - Insufficient permissions to create documents  
      - API quota exceeded  
    - Notes: Google OAuth credentials must be configured.

#### 1.5 Notification Dispatch

- **Overview:**  
  Posts a formatted message to a specified Slack channel announcing the availability of new meeting notes with a direct link.

- **Nodes Involved:**  
  - Post to Slack

- **Node Details:**  
  - **Post to Slack**  
    - Type: Slack node (message posting)  
    - Configuration:  
      - Channel set to `#team-meetings` (default)  
      - Message text includes meeting topic, host, duration, and Google Docs link using templated expressions referencing previous nodes’ outputs.  
      - Authentication via OAuth2  
    - Key Expressions:  
      - `{{ $('Normalize Data').item.json.topic }}`, `{{ $('Normalize Data').item.json.host }}`, `{{ $('Normalize Data').item.json.duration }}`, and `{{ $('Save to Google Docs').item.json.webViewLink }}`  
    - Inputs: Output from Save to Google Docs  
    - Outputs: None (terminal node)  
    - Version: v1  
    - Failure Modes:  
      - Slack OAuth token missing or expired  
      - Channel does not exist or permission denied  
      - Network/API errors  
    - Notes: Requires Slack OAuth credentials and proper channel configuration.

#### 1.6 Setup Guidance

- **Overview:**  
  Provides a sticky note within the workflow containing step-by-step setup instructions for all required external integrations.

- **Nodes Involved:**  
  - Setup Instructions (Sticky Note)

- **Node Details:**  
  - **Setup Instructions**  
    - Type: Sticky Note  
    - Configuration: Markdown content with setup steps for Zoom app, Google Docs OAuth, Slack OAuth, and OpenAI API key.  
    - Inputs: None  
    - Outputs: None  
    - Version: v1  
    - Notes: Serves as an internal documentation aid for workflow maintainers.

---

### 3. Summary Table

| Node Name              | Node Type         | Functional Role          | Input Node(s)         | Output Node(s)        | Sticky Note                                                                                               |
|------------------------|-------------------|-------------------------|-----------------------|-----------------------|-----------------------------------------------------------------------------------------------------------|
| Zoom Meeting Webhook    | Webhook           | Input Reception         | -                     | Normalize Data        | Refer to Setup Instructions for Zoom app webhook setup                                                   |
| Normalize Data         | Code              | Data Normalization      | Zoom Meeting Webhook  | Generate AI Notes     | Contains JS code for extracting and normalizing Zoom event data                                          |
| Generate AI Notes      | OpenAI            | AI Processing           | Normalize Data        | Save to Google Docs   | Uses GPT-4 for accurate meeting note summaries                                                           |
| Save to Google Docs    | Google Docs       | Document Storage        | Generate AI Notes     | Post to Slack         | Saves notes with dynamic title; Google OAuth required                                                   |
| Post to Slack          | Slack             | Notification Dispatch   | Save to Google Docs   | -                     | Posts formatted meeting notes link to Slack channel; Slack OAuth required                               |
| Setup Instructions     | Sticky Note       | Setup Guidance          | -                     | -                     | ## 🛠️ Setup Steps<br>Zoom, Google Docs, Slack, OpenAI configuration details (see section 5 for full text) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the webhook node to receive Zoom meeting ended events:**  
   - Add a **Webhook** node named `Zoom Meeting Webhook`.  
   - Set HTTP Method to `POST`.  
   - Set the path to `zoom-meeting-ended`.  
   - Ensure the node is active and publicly reachable (via n8n cloud or tunnel).  
   - Copy the webhook URL for Zoom app configuration.

2. **Create a Code node to normalize Zoom payload:**  
   - Add a **Code** node named `Normalize Data`.  
   - Use the following JavaScript to extract data:  
     ```javascript
     const e = $input.first().json;

     return {
       json: {
         meeting_id: e.payload.object.id,
         topic: e.payload.object.topic,
         host: e.payload.object.host_email,
         start_time: e.payload.object.start_time,
         end_time: e.payload.object.end_time,
         duration: e.payload.object.duration,
         participants: e.payload.object.participant_count || 0,
         transcript: e.payload.object.transcript || '',
         timestamp: new Date().toISOString()
       }
     };
     ```
   - Connect `Zoom Meeting Webhook` output to this node input.

3. **Add OpenAI node to generate meeting notes:**  
   - Add an **OpenAI** node named `Generate AI Notes`.  
   - Set resource to `Chat` and operation to `create`.  
   - Use GPT-4 model via OpenAI API key configured in credentials.  
   - Construct prompt dynamically from normalized data (e.g., inject `transcript`, `topic`, etc. as message content).  
   - Connect `Normalize Data` output to this node input.

4. **Add Google Docs node to save notes:**  
   - Add a **Google Docs** node named `Save to Google Docs`.  
   - Set action to create a new document.  
   - Set title as `Meeting Notes - {{ $json.topic }} - {{ $json.timestamp }}` using expressions.  
   - Use the AI-generated notes text as document content.  
   - Connect `Generate AI Notes` output to this node input.  
   - Configure Google OAuth credentials with permissions to create/edit documents.

5. **Add Slack node to post notification:**  
   - Add a **Slack** node named `Post to Slack`.  
   - Set channel to `#team-meetings` (or your preferred channel).  
   - Use OAuth2 credentials for Slack.  
   - Construct the message text with template expressions referencing:  
     - Meeting topic: `{{ $('Normalize Data').item.json.topic }}`  
     - Host: `{{ $('Normalize Data').item.json.host }}`  
     - Duration: `{{ $('Normalize Data').item.json.duration }}` mins  
     - Google Docs link: `{{ $('Save to Google Docs').item.json.webViewLink }}`  
   - Connect `Save to Google Docs` output to this node input.

6. **Add a Sticky Note for setup instructions:**  
   - Add a **Sticky Note** node named `Setup Instructions`.  
   - Paste the following markdown content:  
     ```
     ## 🛠️ Setup Steps
     ### 1. Zoom App
     - Go to Zoom Developer Console → create App.  
     - Enable event **`meeting.ended`**.  
     - Paste workflow webhook URL.

     ### 2. Google Docs
     - Connect Google OAuth in n8n.  
     - Docs auto-saved in your Google Drive.

     ### 3. Slack
     - Connect Slack OAuth.  
     - Replace channel `#team-meetings`.

     ### 4. OpenAI
     - Add your OpenAI API key.  
     - Uses **GPT-4** for accurate summaries.

     ---
     ```
   - Place it clearly for reference.

7. **Connect nodes in order:**  
   - `Zoom Meeting Webhook` → `Normalize Data` → `Generate AI Notes` → `Save to Google Docs` → `Post to Slack`

8. **Credentials Setup:**  
   - Configure OAuth2 for Google Docs with scopes for document creation/editing.  
   - Configure OAuth2 for Slack with permission to post messages in target channel.  
   - Configure OpenAI API key with GPT-4 access.

9. **Test the workflow:**  
   - Generate a Zoom meeting ended event (real or simulated) to trigger the webhook.  
   - Verify AI notes generation, Google Doc creation, and Slack notification.

---

### 5. General Notes & Resources

| Note Content                                                                                                      | Context or Link                                                       |
|------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------|
| Workflow uses GPT-4 model for improved accuracy and detail in meeting notes.                                     | Requires OpenAI GPT-4 API key                                        |
| Google Docs created are saved automatically in the authenticated user’s Google Drive.                            | Google OAuth configured in n8n                                       |
| Slack channel default is `#team-meetings`; modify this in the Slack node to your team’s preferred channel.       | Slack OAuth2 credentials required                                    |
| Zoom app must be configured to send `meeting.ended` events to the webhook URL provided by this workflow.        | Zoom Developer Console setup                                         |
| Setup instructions included as a sticky note inside the workflow for quick reference by maintainers.             | See node `Setup Instructions`                                        |
| Ensure n8n instance is publicly accessible or tunneled to receive Zoom webhook calls.                            | Network and security consideration                                   |

---

**Disclaimer:**  
The provided text is extracted exclusively from an automated workflow built with n8n, a workflow automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.