Automated Lead Qualification Voice Agent with OpenAI GPT, Twilio, Elevenlabs and Gmail

https://n8nworkflows.xyz/workflows/automated-lead-qualification-voice-agent-with-openai-gpt--twilio--elevenlabs-and-gmail-9862


# Automated Lead Qualification Voice Agent with OpenAI GPT, Twilio, Elevenlabs and Gmail

### 1. Workflow Overview

This workflow automates lead qualification via voice calls using a combination of Google Sheets, OpenAI GPT, Twilio (via ElevenLabs API), and Gmail. It targets sales and customer success teams seeking to streamline the process of engaging potential leads, capturing call insights, and scheduling follow-ups.

The workflow is logically divided into three main blocks:

- **1.1 Initiate Call:** Detects new leads from a Google Sheet and initiates an outbound voice call with a personalized AI-generated opener.
- **1.2 Fetch Client Data:** Retrieves detailed lead information from Google Sheets based on phone number to personalize interactions.
- **1.3 Outbound Call Processing:** Handles webhook callbacks after calls, processes call transcripts with AI to extract insights and lead qualification status, updates lead records, and optionally sends follow-up emails.

---

### 2. Block-by-Block Analysis

#### 2.1 Initiate Call

- **Overview:**  
  This block triggers on new lead entries in Google Sheets, generates a personalized call opener using OpenAI, and initiates an outbound call via ElevenLabs/Twilio.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Outbound Call  
  - Look up Customer Data  
  - Message a model1 (OpenAI for call opener)  
  - Respond to Webhook  
  - Edit Fields1  

- **Node Details:**

  - **Google Sheets Trigger**  
    - Type: Trigger node for Google Sheets changes  
    - Configuration: Watches "Form Responses 1" sheet for new rows every minute  
    - Inputs: None (trigger)  
    - Outputs: New lead data including phone number and service request  
    - Failure modes: OAuth token expiry, sheet access issues  

  - **Outbound Call**  
    - Type: HTTP Request node  
    - Configuration: POST to ElevenLabs’ Twilio outbound call API with agent ID, phone number, and agent phone number ID  
    - Inputs: Phone number from Google Sheets Trigger  
    - Outputs: Initiates outbound call, no direct output consumed downstream  
    - Failure modes: API errors, authentication failure, invalid phone number format  

  - **Look up Customer Data**  
    - Type: Google Sheets node (read)  
    - Configuration: Queries sheet filtering by "Phone Number (Please enter + country code)" matching called number from webhook  
    - Inputs: Called number from webhook data (via Edit Fields1)  
    - Outputs: Lead details (e.g., First Name, Email, Service Request)  
    - Failure modes: No matching row found, permission errors  

  - **Message a model1**  
    - Type: OpenAI GPT node (Langchain wrapper)  
    - Configuration: Uses GPT-4O-MINI to generate a personalized voice opener message referencing lead’s first name and service request  
    - Inputs: Lead data from Look up Customer Data  
    - Outputs: JSON with key "Voice Opener" string  
    - Failure modes: API key issues, rate limiting, prompt errors  

  - **Respond to Webhook**  
    - Type: Respond to Webhook node  
    - Configuration: Responds with JSON containing the generated "Voice Opener" from OpenAI output  
    - Inputs: Message a model1 output  
    - Outputs: HTTP response to webhook caller  
    - Failure modes: Output missing expected data  

  - **Edit Fields1**  
    - Type: Set node  
    - Configuration: Extracts country code and local phone number from webhook’s called_number field using regex  
    - Inputs: Webhook payload  
    - Outputs: Adds fields "country_code" and "phone_number" for downstream use  
    - Failure modes: Unexpected phone number formatting  

---

#### 2.2 Fetch Client Data

- **Overview:**  
  This block processes incoming webhook data after call completion, cleans the transcript, and prepares data for AI analysis.

- **Nodes Involved:**  
  - Webhook1  
  - Code  

- **Node Details:**

  - **Webhook1**  
    - Type: Webhook node (POST method)  
    - Configuration: Receives post-call data including transcript and call metadata from ElevenLabs/Twilio webhook  
    - Inputs: None (trigger)  
    - Outputs: Raw call data, transcript array, call status  
    - Failure modes: Webhook signature verification failure, payload format changes  

  - **Code**  
    - Type: Function (JavaScript)  
    - Configuration: Parses transcript array, capitalizes roles, and joins messages into a clean multiline string for AI consumption  
    - Inputs: Webhook1 JSON data  
    - Outputs: JSON with "clean_transcript" string  
    - Edge cases: Transcript missing or empty, malformed data  

---

#### 2.3 Outbound Call Processing

- **Overview:**  
  This block interprets the call transcript with OpenAI GPT to extract lead qualification data, updates Google Sheets with call insights, and sends follow-up emails when appropriate.

- **Nodes Involved:**  
  - Message a model  
  - If  
  - Edit Fields  
  - Merge  
  - Get row(s) in sheet1  
  - Update Call Data  
  - Send Schedule Request  

- **Node Details:**

  - **Message a model**  
    - Type: OpenAI GPT node  
    - Configuration: Uses GPT-4O-MINI with a system prompt instructing AI to analyze call transcript and output a strict JSON containing call summary, lead status, project details, budget, timeline, and pain points  
    - Inputs: Clean transcript from Code node  
    - Outputs: JSON object with structured lead qualification data  
    - Edge cases: AI returning malformed JSON, timeout, invalid transcript  

  - **If**  
    - Type: Conditional node  
    - Configuration: Checks if leadStatus field in AI response equals "Qualified"  
    - Inputs: Message a model output  
    - Outputs: Conditional routing for qualified leads (true) or other statuses (false)  
    - Failure modes: Missing leadStatus field, case sensitivity issues  

  - **Edit Fields**  
    - Type: Set node  
    - Configuration: Extracts country code and phone number from webhook call metadata for updating sheet  
    - Inputs: Webhook1 data  
    - Outputs: Fields "country_code" and "phone_number" formatted for sheet update  
    - Edge cases: Phone number regex mismatch  

  - **Merge**  
    - Type: Merge node combining two input streams by position  
    - Configuration: Combines AI analysis output and Google Sheets data from Get row(s) in sheet1  
    - Inputs: Message a model output and Get row(s) output  
    - Outputs: Combined data for updating call records  
    - Edge cases: Mismatched array lengths, missing data  

  - **Get row(s) in sheet1**  
    - Type: Google Sheets read node  
    - Configuration: Fetches rows matching called number from webhook dynamic variables for data update  
    - Inputs: Merged data from Merge node  
    - Outputs: Lead row info for update  
    - Edge cases: No matching rows found  

  - **Update Call Data**  
    - Type: Google Sheets update/append node  
    - Configuration: Updates or appends lead row with AI call summary, lead status, transcript, budget, timeline, and phone number  
    - Inputs: Merged data and dynamic variables from webhook  
    - Outputs: Updated Google Sheet record  
    - Failure modes: Sheet access errors, write conflicts  

  - **Send Schedule Request**  
    - Type: Gmail send email node  
    - Configuration: Sends a follow-up email to lead’s email with a calendar link for scheduling discovery session  
    - Inputs: Lead email and first name from Google Sheets row  
    - Outputs: Email sent confirmation  
    - Failure modes: SMTP auth failure, invalid email address  

---

### 3. Summary Table

| Node Name            | Node Type                         | Functional Role                           | Input Node(s)           | Output Node(s)            | Sticky Note                                                                                              |
|----------------------|----------------------------------|-----------------------------------------|------------------------|---------------------------|--------------------------------------------------------------------------------------------------------|
| Google Sheets Trigger | n8n-nodes-base.googleSheetsTrigger | Detect new leads from Google Sheets     | None                   | Outbound Call             | ## 1. Initiate Call ##                                                                                  |
| Outbound Call         | n8n-nodes-base.httpRequest        | Initiate outbound call via ElevenLabs/Twilio | Google Sheets Trigger | Edit Fields1              | ## 1. Initiate Call ##                                                                                  |
| Edit Fields1          | n8n-nodes-base.set                | Parse country code and phone number from webhook called_number | Webhook                 | Look up Customer Data     | ## 1. Initiate Call ##                                                                                  |
| Look up Customer Data | n8n-nodes-base.googleSheets       | Fetch lead details by phone number      | Edit Fields1            | Message a model1          | ## 1. Initiate Call ##                                                                                  |
| Message a model1      | @n8n/n8n-nodes-langchain.openAi  | Generate personalized voice opener      | Look up Customer Data   | Respond to Webhook        | ## 1. Initiate Call ##                                                                                  |
| Respond to Webhook    | n8n-nodes-base.respondToWebhook   | Send voice opener to webhook caller     | Message a model1        | None                     | ## 1. Initiate Call ##                                                                                  |
| Webhook               | n8n-nodes-base.webhook            | Receive webhook call initiation data    | None                   | Edit Fields1              | ## 2. Fetch Client Data ##                                                                               |
| Webhook1              | n8n-nodes-base.webhook            | Receive post-call data and transcript   | None                   | Code                      | ## 2. Fetch Client Data ##                                                                               |
| Code                 | n8n-nodes-base.code               | Format transcript string for AI input   | Webhook1                | Message a model           | ## 2. Fetch Client Data ##                                                                               |
| Message a model       | @n8n/n8n-nodes-langchain.openAi  | Analyze transcript, extract lead info   | Code                    | If, Edit Fields           | ## 3. Outbound Call Processing ##                                                                        |
| If                   | n8n-nodes-base.if                 | Check if lead is qualified               | Message a model         | Merge (true path)         | ## 3. Outbound Call Processing ##                                                                        |
| Edit Fields           | n8n-nodes-base.set                | Extract country code and phone number   | Message a model (false) | Update Call Data, Merge   | ## 3. Outbound Call Processing ##                                                                        |
| Merge                 | n8n-nodes-base.merge              | Combine AI analysis and Google Sheets data | If                     | Get row(s) in sheet1      | ## 3. Outbound Call Processing ##                                                                        |
| Get row(s) in sheet1  | n8n-nodes-base.googleSheets       | Fetch lead row for update                 | Merge                   | Send Schedule Request     | ## 3. Outbound Call Processing ##                                                                        |
| Update Call Data      | n8n-nodes-base.googleSheets       | Update lead record with call insights    | Edit Fields, Merge      | None                     | ## 3. Outbound Call Processing ##                                                                        |
| Send Schedule Request | n8n-nodes-base.gmail              | Send follow-up scheduling email          | Get row(s) in sheet1    | None                     | ## 3. Outbound Call Processing ##                                                                        |
| Sticky Note           | n8n-nodes-base.stickyNote         | Workflow documentation and block headers | None                   | None                     | ## What is this? (General workflow explanation and notes)                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node: Google Sheets Trigger**  
   - Type: Google Sheets Trigger  
   - Set Event to "Row Added"  
   - Configure to watch sheet "Form Responses 1" in your Google Sheet document  
   - Poll every minute  
   - Authenticate with Google Sheets OAuth2 credentials  

2. **Create Outbound Call Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: https://api.elevenlabs.io/v1/convai/twilio/outbound-call  
   - Body Parameters:  
     - agent_id: `agent_0501k5kszqfxe65byc7m2ebkg498`  
     - agent_phone_number_id: `phnum_1201k5ksx9cmehgah78rf8es27jf`  
     - to_number: Use phone number from Google Sheets Trigger (field "Phone Number (Please enter + country code)")  
   - Add HTTP Header Authentication using your ElevenLabs personal credentials  
   - Connect output of Google Sheets Trigger to this node  

3. **Create Webhook Node (Webhook)**  
   - Type: Webhook (POST)  
   - Path: unique identifier (e.g., "030f0967-0d16-43db-aa72-ad2d5909f945")  
   - This webhook will receive call initiation data (optional depending on your integration)  

4. **Create Edit Fields1 Node**  
   - Type: Set  
   - Extract "country_code" via regex from webhook JSON field `body.called_number`  
   - Extract "phone_number" by removing country code from `body.called_number`  
   - Pass all other fields through  
   - Connect Webhook output to this node  

5. **Create Look up Customer Data Node**  
   - Type: Google Sheets  
   - Operation: Read rows  
   - Filter by "Phone Number (Please enter + country code)" equal to incoming phone number from Edit Fields1  
   - Use same Google Sheet and credentials as trigger  
   - Connect Edit Fields1 output here  

6. **Create Message a model1 Node**  
   - Type: OpenAI GPT (Langchain)  
   - Model: GPT-4O-MINI  
   - Prompt: "Write a friendly, concise outbound call opener using lead’s first name and service request."  
   - Input variables: First Name and Service Request from Look up Customer Data  
   - Output JSON with key "Voice Opener"  
   - Connect Look up Customer Data output here  

7. **Create Respond to Webhook Node**  
   - Type: Respond to Webhook  
   - Respond with JSON containing "voiceopener" from Message a model1 output  
   - Connect Message a model1 output here  

8. **Create Webhook1 Node**  
   - Type: Webhook (POST)  
   - Path: unique identifier for post-call data (e.g., "37e8d818-8265-43e9-803e-00119da5f3cb")  
   - This webhook receives call transcript and metadata from ElevenLabs/Twilio  

9. **Create Code Node**  
   - Type: Function  
   - JavaScript: Parses transcript array from Webhook1 JSON, capitalizes roles, joins messages with newline into "clean_transcript" string  
   - Connect Webhook1 output here  

10. **Create Message a model Node (AI Call Analysis)**  
    - Type: OpenAI GPT (Langchain)  
    - Model: GPT-4O-MINI  
    - System prompt: Instruct AI to analyze transcript and output strict JSON with fields: callSummary, leadStatus, projectDetails, budget, timeline, painPoints  
    - Input: "clean_transcript" from Code node  
    - Connect Code output here  

11. **Create If Node**  
    - Type: If condition  
    - Condition: Check if `leadStatus == "Qualified"` from AI output  
    - Connect Message a model output here  

12. **Create Edit Fields Node**  
    - Type: Set  
    - Extract "country_code" and "phone_number" from Webhook1 call metadata (external_number)  
    - Connect Message a model output here (false branch)  

13. **Create Merge Node**  
    - Type: Merge (combine by position)  
    - Connect If node true branch to input 1  
    - Connect Edit Fields node to input 2  

14. **Create Get row(s) in sheet1 Node**  
    - Type: Google Sheets  
    - Filter: "Phone Number (Please enter + country code)" equals the called number from webhook dynamic variables  
    - Connect Merge node output here  

15. **Create Update Call Data Node**  
    - Type: Google Sheets  
    - Operation: Append or Update row  
    - Map fields for Budget, Timeline, Transcript, Lead status, Call summary, Phone Number from AI output and webhook variables  
    - Connect Edit Fields and Merge node outputs as inputs  

16. **Create Send Schedule Request Node**  
    - Type: Gmail Send Email node  
    - To: Lead's email from Google Sheets row  
    - Subject: "Thinkspire Meeting"  
    - Body: Friendly email with calendar scheduling link  
    - Connect Get row(s) in sheet1 output here  

17. **Create Sticky Notes (optional)**  
    - Add sticky notes to document workflow blocks and usage instructions as in the original workflow  

18. **Configure Credentials**  
    - Google Sheets OAuth2 with access to your sheet  
    - OpenAI API key with GPT-4O-MINI model access  
    - Gmail OAuth2 for sending emails  
    - ElevenLabs API key / Twilio credentials for outbound calls  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                   | Context or Link                                                                                             |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| This workflow automates lead qualification by integrating voice call handling, AI transcript analysis, and email follow-ups. It is designed for sales teams to streamline lead engagement and qualification.                                                     | Workflow purpose and high-level overview                                                                   |
| Prerequisites: OpenAI, Google Sheets, Twilio, ElevenLabs setup with imported Twilio number, configured agent, prompts, and webhooks.                                                                                                                         | Setup instructions                                                                                          |
| Customization options include switching the data source, modifying AI prompts for call openers, changing telephony provider integrations, and adding notification channels.                                                                                     | Adaptability notes                                                                                          |
| Official ElevenLabs API documentation: https://elevenlabs.io/docs/api-reference                                                                                                                                                                               | ElevenLabs API reference                                                                                    |
| Use Google Sheets for CRM data management, ensuring that phone numbers include country codes in the specified format to avoid matching errors.                                                                                                               | Data source best practices                                                                                  |
| AI prompt in "Message a model" enforces JSON-only output for reliable parsing downstream. Ensure prompt integrity to avoid parsing errors.                                                                                                                  | AI prompt design tip                                                                                        |
| Phone number regex extraction assumes E.164 format with a leading plus sign and 1-2 digit country code. Modify regex if your phone number format differs.                                                                                                    | Regex extraction note                                                                                       |
| The workflow uses webhook nodes for real-time interaction with ElevenLabs/Twilio callbacks, enabling immediate processing of call results and transcripts.                                                                                                   | Integration pattern                                                                                         |
| Email follow-up includes a calendar link for easy scheduling of discovery sessions, enhancing lead conversion chances.                                                                                                                                         | Follow-up strategy                                                                                          |
| Sticky notes in the workflow provide block-level documentation for maintainers and collaborators.                                                                                                                                                              | Internal documentation aid                                                                                  |

---

**Disclaimer:** The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. The processing strictly respects current content policies and contains no illegal, offensive, or protected content. All data processed is legal and public.