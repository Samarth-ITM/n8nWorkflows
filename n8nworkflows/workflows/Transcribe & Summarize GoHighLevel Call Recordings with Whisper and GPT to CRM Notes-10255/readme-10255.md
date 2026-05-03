Transcribe & Summarize GoHighLevel Call Recordings with Whisper and GPT to CRM Notes

https://n8nworkflows.xyz/workflows/transcribe---summarize-gohighlevel-call-recordings-with-whisper-and-gpt-to-crm-notes-10255


# Transcribe & Summarize GoHighLevel Call Recordings with Whisper and GPT to CRM Notes

### 1. Workflow Overview

This workflow automates the transcription and summarization of GoHighLevel (GHL) call recordings using AI models (OpenAI Whisper for transcription and GPT for summarization) and posts the summarized notes back into the related CRM contact record within GHL.

**Target Use Cases:**  
- Sales and acquisitions teams needing fast, accurate call summaries in CRM.  
- Real estate or service businesses logging client interactions automatically.  
- QA and training teams requiring searchable call histories.  
- Agencies integrating AI-generated call insights directly into GHL contacts.

**Logical Blocks:**  
- **1.1 Input Reception:** Receive webhook event from GHL when a call ends.  
- **1.2 Delay:** Wait to ensure recording availability.  
- **1.3 Data Extraction:** Extract contact and call metadata from webhook payload.  
- **1.4 Conversation & Message Retrieval:** Retrieve conversation ID, message IDs, and download the audio recording from GHL API.  
- **1.5 AI Processing:** Transcribe audio recording to text and summarize using GPT.  
- **1.6 Formatting Summary:** Format AI-generated summary into structured JSON for GHL API.  
- **1.7 Update CRM:** Post the summary note into the relevant GHL contact’s notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  The workflow starts by receiving a webhook POST from GoHighLevel when a call is completed. This node captures essential call and contact data.

- **Nodes Involved:**  
  - `Call Ended` (Webhook)  
  - `Wait 60 seconds` (Delay)  
  - `Edit Fields1` (Set Fields)

- **Node Details:**  
  - **Call Ended**  
    - Type: Webhook  
    - Role: Entry point triggered by GHL’s call completion webhook.  
    - Config: HTTP POST method on path `/sample`.  
    - Input: HTTP POST from GHL.  
    - Output: JSON with call, contact, and conversation metadata.  
    - Edge Cases: Webhook URL misconfiguration, missing payload data.  
  - **Wait 60 seconds**  
    - Type: Wait  
    - Role: Ensures the call recording is fully processed and available in GHL before fetching.  
    - Config: Fixed 60 seconds delay (modifiable).  
    - Input: Output from webhook node.  
    - Output: Delayed data pass-through.  
    - Edge Cases: Longer processing times may require increased delay.  
  - **Edit Fields1**  
    - Type: Set  
    - Role: Extracts and sets key variables (`full_name`, `phone`, `contact_id`) from webhook data for downstream use.  
    - Config: Uses expressions to pull data from the webhook payload.  
    - Input: Output from delay node.  
    - Output: JSON with clean variable assignments.  
    - Edge Cases: Missing or malformed webhook payload fields.

#### 2.2 Conversation & Message Retrieval

- **Overview:**  
  This block fetches the conversation associated with the contact, retrieves message IDs related to the call, and downloads the audio recording file from GHL.

- **Nodes Involved:**  
  - `GHL Get Conversation ID` (HTTP Request)  
  - `GHL Get Message ID` (HTTP Request)  
  - `GHL Get Audio` (HTTP Request)

- **Node Details:**  
  - **GHL Get Conversation ID**  
    - Type: HTTP Request  
    - Role: Queries GHL API for conversations filtered by `contact_id`.  
    - Config: GET request to `/conversations/search` with query param `contactId` from `Edit Fields1`.  
    - Headers include versioning info for GHL API.  
    - Input: Output from `Edit Fields1`.  
    - Output: JSON containing conversation list.  
    - Edge Cases: API rate limits, invalid contact ID, network errors.  
  - **GHL Get Message ID**  
    - Type: HTTP Request  
    - Role: Retrieves messages of type `TYPE_CALL` for the first conversation found.  
    - Config: GET request to `/conversations/{{conversationId}}/messages` with query param `type=TYPE_CALL`.  
    - Input: Output from `GHL Get Conversation ID`.  
    - Output: JSON with call message details including recording location.  
    - Edge Cases: Empty conversations, message type filtering failing, API errors.  
  - **GHL Get Audio**  
    - Type: HTTP Request  
    - Role: Downloads the audio recording file for the call message.  
    - Config: GET request to `/conversations/messages/{{messageId}}/locations/{{locationId}}/recording` with appropriate headers.  
    - Input: Output from `GHL Get Message ID`.  
    - Output: Audio file binary data for transcription.  
    - Edge Cases: Recording not yet available, permission or auth errors, invalid message/location IDs.

#### 2.3 AI Processing (Transcription & Summarization)

- **Overview:**  
  This block sends the audio recording to OpenAI Whisper for transcription, then summarizes the transcript into concise bullet points using GPT-4o-mini.

- **Nodes Involved:**  
  - `Transcribe a recording` (OpenAI Audio Transcription)  
  - `Message a model` (OpenAI GPT Summarization)

- **Node Details:**  
  - **Transcribe a recording**  
    - Type: OpenAI (LangChain node)  
    - Role: Sends audio binary data to Whisper API for speech-to-text transcription.  
    - Config: Operation set to `transcribe` audio resource, uses OpenAI API credentials.  
    - Input: Audio file from `GHL Get Audio`.  
    - Output: JSON transcript text.  
    - Edge Cases: Audio format issues, API timeout, auth failure.  
  - **Message a model**  
    - Type: OpenAI (LangChain node)  
    - Role: Summarizes the transcript into 5–8 bullet points focused on sales/real estate call notes.  
    - Config: Uses `gpt-4o-mini` model with a carefully crafted prompt emphasizing fact extraction, no assumptions, and specific summary rules.  
    - Input: Transcript text from `Transcribe a recording`.  
    - Output: Summarized bullet points text.  
    - Edge Cases: Prompt failure, API rate limits, model errors.

#### 2.4 Formatting Summary

- **Overview:**  
  Converts the plain text summary from GPT into structured JSON matching the GHL API schema for creating a contact note.

- **Nodes Involved:**  
  - `Code` (JavaScript Code node)

- **Node Details:**  
  - Type: Code (JavaScript)  
  - Role: Extracts AI-generated content, normalizes newlines, trims whitespace, and constructs JSON body with userId and note text for GHL API.  
  - Configuration: Uses a default userId if none provided; throws errors if no content found.  
  - Input: Output from `Message a model`.  
  - Output: JSON object with fields `userId` and `body` ready for GHL API.  
  - Edge Cases: Missing userId or summary content, malformed AI output.

#### 2.5 Update CRM Contact Notes

- **Overview:**  
  Posts the structured summary note back to the contact’s notes in GHL via API.

- **Nodes Involved:**  
  - `Add note to contact` (HTTP Request)

- **Node Details:**  
  - Type: HTTP Request  
  - Role: Sends a POST request to GHL API adding the summary note to the contact record.  
  - Config: POST to `/contacts/{{contact_id}}/notes` with JSON body from `Code` node. Includes required headers and API version.  
  - Input: JSON from `Code` node, contact ID from webhook.  
  - Output: API response confirming note creation.  
  - Edge Cases: API auth errors, invalid contact ID, network failures.

---

### 3. Summary Table

| Node Name             | Node Type                      | Functional Role                         | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                                      |
|-----------------------|--------------------------------|---------------------------------------|-----------------------|-----------------------|------------------------------------------------------------------------------------------------------------------|
| Call Ended            | Webhook                        | Entry point receiving GHL webhook     | —                     | Wait 60 seconds       | —                                                                                                                |
| Wait 60 seconds       | Wait                          | Delay to ensure recording availability| Call Ended            | Edit Fields1          | —                                                                                                                |
| Edit Fields1          | Set                           | Extract key fields from webhook data  | Wait 60 seconds       | GHL Get Conversation ID| —                                                                                                                |
| GHL Get Conversation ID| HTTP Request                  | Retrieve conversation by contact ID   | Edit Fields1          | GHL Get Message ID     | "Find the conversation and get the call recording" with GHL OAuth setup instructions                             |
| GHL Get Message ID    | HTTP Request                  | Retrieve call message IDs from conversation | GHL Get Conversation ID | GHL Get Audio          | —                                                                                                                |
| GHL Get Audio         | HTTP Request                  | Download audio recording file          | GHL Get Message ID     | Transcribe a recording | —                                                                                                                |
| Transcribe a recording| OpenAI Audio Transcription     | Transcribe audio to text               | GHL Get Audio          | Message a model        | "AI transcribes and summarizes" — prompt can be customized for business needs                                   |
| Message a model       | OpenAI GPT Summarization       | Summarize transcript into bullet points | Transcribe a recording | Code                   | —                                                                                                                |
| Code                  | Code (JavaScript)              | Format summary into JSON for GHL API  | Message a model        | Add note to contact    | —                                                                                                                |
| Add note to contact   | HTTP Request                  | Post summary note to GHL contact notes | Code                   | —                      | "Find and update GHL contact notes" — adds summary as a note                                                    |
| Sticky Note           | Sticky Note                   | Documentation and instructions         | —                     | —                      | Describes overall workflow purpose, use cases, and how it works                                                |
| Sticky Note1          | Sticky Note                   | GHL OAuth credential setup instructions | —                     | —                      | "Find the conversation and get the call recording" detailed note                                               |
| Sticky Note2          | Sticky Note                   | AI transcription and summarization setup | —                     | —                      | Notes on AI account connection and prompt customization                                                        |
| Sticky Note3          | Sticky Note                   | Notes on posting summary to GHL contact | —                     | —                      | Explains purpose of adding note to contact                                                                    |
| Sticky Note4          | Sticky Note                   | Instructions for setting up GHL webhook | —                     | —                      | Steps to configure GHL workflow and webhook trigger                                                            |
| Sticky Note5          | Sticky Note                   | Credentials setup for AI and GHL OAuth | —                     | —                      | Detailed instructions for setting up OpenAI and GoHighLevel OAuth credentials, with video link                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Name: `Call Ended`  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `/sample`  
   - Purpose: Receive the GHL call completion webhook.

2. **Add Wait Node**  
   - Name: `Wait 60 seconds`  
   - Type: Wait  
   - Duration: 60 seconds  
   - Connect: Output of `Call Ended` → Input of `Wait 60 seconds`.

3. **Add Set Node**  
   - Name: `Edit Fields1`  
   - Type: Set  
   - Parameters: Create fields  
     - `full_name` = Expression: `{{$json["body"]["full_name"]}}` from `Call Ended`  
     - `phone` = Expression: `{{$json["body"]["phone"]}}`  
     - `contact_id` = Expression: `{{$json["body"]["contact_id"]}}`  
   - Connect: Output of `Wait 60 seconds` → Input of `Edit Fields1`.

4. **Add HTTP Request Node**  
   - Name: `GHL Get Conversation ID`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://services.leadconnectorhq.com/conversations/search`  
   - Query Parameter: `contactId` = Expression: `{{$json["contact_id"]}}` from `Edit Fields1`  
   - Headers: Add header `Version: 2021-07-28`  
   - Connect: Output of `Edit Fields1` → Input of `GHL Get Conversation ID`.

5. **Add HTTP Request Node**  
   - Name: `GHL Get Message ID`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://services.leadconnectorhq.com/conversations/{{$json["conversations"][0]["id"]}}/messages`  
   - Query Parameter: `type=TYPE_CALL`  
   - Headers: Add header `Version: 2021-07-28`  
   - Connect: Output of `GHL Get Conversation ID` → Input of `GHL Get Message ID`.

6. **Add HTTP Request Node**  
   - Name: `GHL Get Audio`  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://services.leadconnectorhq.com/conversations/messages/{{$json["messages"]["messages"][0]["id"]}}/locations/{{$json["messages"]["messages"][1]["locationId"]}}/recording`  
   - Headers: Add header `Version: 2021-07-28`  
   - Connect: Output of `GHL Get Message ID` → Input of `GHL Get Audio`.

7. **Add OpenAI Node for Transcription**  
   - Name: `Transcribe a recording`  
   - Type: OpenAI (LangChain node)  
   - Operation: `transcribe` audio resource  
   - Credentials: Select your configured OpenAI API credential  
   - Connect: Output of `GHL Get Audio` → Input of `Transcribe a recording`.

8. **Add OpenAI Node for Summarization**  
   - Name: `Message a model`  
   - Type: OpenAI (LangChain node)  
   - Model: `gpt-4o-mini`  
   - Messages:  
     - System role: Extractive summarizer instructions emphasizing fact-based bullet points.  
     - User role: Supply transcript text via expression `{{$json["text"]}}` from transcription node.  
   - Credentials: OpenAI API credential  
   - Connect: Output of `Transcribe a recording` → Input of `Message a model`.

9. **Add Code Node to Format Summary**  
   - Name: `Code`  
   - Type: Code (JavaScript)  
   - Purpose: Extract and normalize the summary text, prepare JSON with `userId` and `body` fields.  
   - Use the provided JavaScript logic for content extraction and validation.  
   - Connect: Output of `Message a model` → Input of `Code`.

10. **Add HTTP Request Node to Post Note**  
    - Name: `Add note to contact`  
    - Type: HTTP Request  
    - Method: POST  
    - URL: `https://services.leadconnectorhq.com/contacts/{{$json["userId"]}}/notes` (replace with appropriate contact_id path)  
    - Body Content-Type: JSON  
    - JSON Body: Use the output from `Code` node.  
    - Headers: Add header `Version: 2021-07-28`  
    - Credentials: Use your configured GHL OAuth2 credential  
    - Connect: Output of `Code` → Input of `Add note to contact`.

11. **Credential Setup:**  
    - OpenAI API: Create credential in n8n with your API key for Whisper and GPT.  
    - GoHighLevel OAuth2: Register OAuth app in GHL Agency account, obtain Client ID, Secret, and configure redirect URL in n8n credentials.  
    - Assign these credentials to the respective nodes.

12. **Webhook Configuration in GoHighLevel:**  
    - Create a workflow in GHL with trigger "Call Status" = Completed.  
    - Add a webhook action that POSTs to your n8n webhook URL (`/sample`).  

13. **Testing:**  
    - Make a test call in GHL, ensure webhook triggers, recording is processed, transcript and summary generated, and note added to contact.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                           |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This n8n template automatically transcribes and summarizes GoHighLevel call recordings, adding notes to CRM contacts to eliminate manual note-taking. Ideal for sales, real estate, QA, and agencies using GHL.                                                                                                                                                               | Workflow Overview Sticky Note                                                                             |
| OAuth2 credential setup for GoHighLevel requires Client ID, Secret, and Redirect URL. Refer to GHL Agency settings to create OAuth apps.                                                                                                                                                                                                                                    | Sticky Note5 — OAuth credential setup instructions                                                       |
| Video guidance for OAuth V2 setup: https://www.youtube.com/watch?v=vrbi5TiFJ-g&t=4s                                                                                                                                                                                                                                                                                           | Sticky Note5                                                                                             |
| To set up the GHL webhook: create automation in GHL triggered on Call Status = Completed, add webhook action posting to n8n webhook URL.                                                                                                                                                                                                                                     | Sticky Note4                                                                                             |
| AI prompt is tailored to summarize sales/real estate calls into fact-based, bullet-point CRM notes with explicit rules to avoid assumptions or invented data. Modify prompt as needed for other industries.                                                                                                                                                                    | Sticky Note2                                                                                             |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.