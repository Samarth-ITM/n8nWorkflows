Create AI Personalized Video & Voice Outreach with HeyGen, ElevenLabs & Perplexity

https://n8nworkflows.xyz/workflows/create-ai-personalized-video---voice-outreach-with-heygen--elevenlabs---perplexity-11326


# Create AI Personalized Video & Voice Outreach with HeyGen, ElevenLabs & Perplexity

### 1. Workflow Overview

This workflow automates the creation of hyper-personalized outreach assets—videos, voice notes, and emails—for sales development representatives (SDRs) based on new leads added in a Google Sheet. It is designed for influencer marketing agencies or similar sales teams aiming to scale personalized outreach using multiple AI services.

**Target Use Cases:**  
- Automated lead intake and research  
- AI-driven personalized scriptwriting for outreach  
- Generation of custom AI avatar videos and voice notes  
- Automated delivery of outreach assets via email and optional SMS/WhatsApp  

**Logical Blocks:**

- **1.1 Lead Intake & Extraction:** Detect new leads in Google Sheets and extract the latest row.  
- **1.2 AI Research & Scriptwriting:** Use AI agents to research the lead and company (via Perplexity), then create a personalized outreach script.  
- **1.3 Video Generation:** Generate a personalized avatar video using HeyGen API based on the script.  
- **1.4 Voice Note Generation & Upload:** Convert the script to speech with ElevenLabs, upload audio to Google Drive, and optionally send SMS/WhatsApp with the link.  
- **1.5 Email Generation & Delivery:** Create a personalized outreach email referencing the video and voice note, then send it via Gmail.  
- **1.6 Polling & Control Flow:** Wait for video processing and then proceed with email generation.  

---

### 2. Block-by-Block Analysis

#### 1.1 Lead Intake & Extraction

- **Overview:** Watches for new leads added to a Google Sheet, extracts only the most recent row for processing.  
- **Nodes Involved:** Google Sheets Trigger, Code in JavaScript  
- **Node Details:**

  - **Google Sheets Trigger**  
    - Type: Trigger (event-driven)  
    - Configuration: Polls every minute for new rows in a specified sheet and tab (placeholders for sheet/tab IDs to be replaced by user)  
    - Input: N/A (trigger event)  
    - Output: New rows data from Google Sheets  
    - Edge Cases:  
      - Missing or invalid sheet/tab ID leads to no trigger  
      - Rate limits or connectivity issues with Google Sheets API  

  - **Code in JavaScript**  
    - Type: Function node  
    - Role: Filters the incoming array of rows to output only the last (most recent) row  
    - Key Logic: Checks if items array is empty, outputs last element  
    - Input: Array of new rows from Google Sheets Trigger  
    - Output: Single-item array containing the latest lead  
    - Edge Cases: Empty input array results in no output  

---

#### 1.2 AI Research & Scriptwriting

- **Overview:** Performs deep AI research on the lead and company using Perplexity, then creates a personalized 30-second outreach script for the video avatar narration.  
- **Nodes Involved:** Research Agent, Message a model in Perplexity, Scripting Agent, OpenAI Chat Model, OpenRouter Chat Model  
- **Node Details:**

  - **Research Agent (LangChain Agent)**  
    - Type: AI agent node with LangChain integration  
    - Configuration: Receives lead details as input text with placeholders populated from the lead row; uses Perplexity as an AI research tool to generate structured insights about the prospect and company.  
    - System Message: Detailed instructions to research person, company, opportunities, and sales triggers with strict output format  
    - Input: Latest lead data (name, email, phone, company, LinkedIn, etc.)  
    - Output: AI-generated research insights in structured text  
    - Dependencies: Uses Perplexity Tool via "Message a model in Perplexity" node  
    - Edge Cases: API rate limits, malformed input data, Perplexity service downtime  

  - **Message a model in Perplexity**  
    - Type: AI tool node  
    - Configuration: Uses "sonar-pro" model to process messages passed from Research Agent  
    - Input: Messages containing the research query  
    - Output: Research results for the agent  
    - Edge Cases: API authentication errors, response timeouts  

  - **Scripting Agent (LangChain Agent)**  
    - Type: AI agent node  
    - Configuration: Receives research insights and writes a personalized 30-second script for AI avatar narration, with tone and content rules (casual, friendly, personalized, no jargon, no filler).  
    - System Message: Extensive prompt describing scriptwriting rules, output format, and prohibited content  
    - Input: Output from Research Agent  
    - Output: Plain text script (ready for video narration)  
    - Edge Cases: AI output format errors, timeout, unexpected content  

  - **OpenAI Chat Model**  
    - Type: Language Model node (OpenAI GPT-5.1)  
    - Role: Supports Research Agent by providing language model capabilities  
    - Input/Output: Linked as AI language model for Research Agent  
    - Edge Cases: API key/auth issues, rate limits  

  - **OpenRouter Chat Model**  
    - Type: Language Model node  
    - Role: Supports Scripting Agent similarly using OpenRouter GPT-5.1  
    - Edge Cases: Same as OpenAI Chat Model  

---

#### 1.3 Video Generation

- **Overview:** Sends the personalized script to HeyGen API to generate a video featuring an AI avatar speaking the script.  
- **Nodes Involved:** Heygen Clone AI Creation, Wait 30s, GET Result  
- **Node Details:**

  - **Heygen Clone AI Creation**  
    - Type: HTTP Request  
    - Configuration: POST request to HeyGen’s video generation API with JSON body specifying avatar, voice, text input, and video dimensions  
    - Authentication: Generic HTTP header auth (API key or token must be provided)  
    - Input: Script text from Scripting Agent  
    - Output: Video generation job response including video ID  
    - Edge Cases: API errors, invalid avatar or voice IDs, rate limits  

  - **Wait 30s**  
    - Type: Wait node  
    - Configuration: Pauses workflow for 30 seconds to allow video processing  
    - Edge Cases: Delay too short/long may affect workflow timing  

  - **GET Result**  
    - Type: HTTP Request  
    - Configuration: GET request to HeyGen video status API to check if video is ready, querying by video_id  
    - Authentication: Same as Heygen Clone node  
    - Input: Video ID from previous node  
    - Output: Video status and final video URL when ready  
    - Edge Cases: Polling may fail if video not ready yet, API errors  

---

#### 1.4 Voice Note Generation & Upload

- **Overview:** Converts the personalized script to speech using ElevenLabs, uploads the audio file to Google Drive, and optionally sends an SMS/WhatsApp message with the audio link via Twilio.  
- **Nodes Involved:** Convert text to speech, Upload file, Send an SMS/MMS/WhatsApp message  
- **Node Details:**

  - **Convert text to speech (ElevenLabs)**  
    - Type: ElevenLabs AI node  
    - Configuration: Converts script text to speech using specified voice ID  
    - Input: Script text from Scripting Agent  
    - Output: Audio file data  
    - Edge Cases: Voice ID invalid, API limits, audio generation failures  

  - **Upload file (Google Drive)**  
    - Type: Google Drive node  
    - Configuration: Uploads audio file to specified Google Drive folder (folder ID placeholder) with file name "Audio"  
    - Input: Audio file from ElevenLabs node  
    - Output: Google Drive file metadata including web link  
    - Edge Cases: Permission errors, invalid folder ID, upload failures  

  - **Send an SMS/MMS/WhatsApp message (Twilio)**  
    - Type: Twilio node  
    - Configuration: Sends SMS/WhatsApp message with file link to specified phone number (placeholders for 'to' and 'from')  
    - Input: Google Drive file webContentLink  
    - Output: Message send confirmation  
    - Edge Cases: Invalid phone numbers, credential issues, API limits  

---

#### 1.5 Email Generation & Delivery

- **Overview:** Generates a personalized outreach email referencing the video and voice note, then sends it via Gmail.  
- **Nodes Involved:** AI Agent, OpenAI Chat Model1, Structured Output Parser, OpenAI Chat Model2, Send a message  
- **Node Details:**

  - **AI Agent (LangChain Agent)**  
    - Type: AI agent node  
    - Configuration: Generates a short, personalized outbound email in strict JSON format with "title" (subject) and "body" referencing the personalized video.  
    - Input: Research insights from Research Agent and video URL from GET Result node  
    - Output: JSON containing subject and email body text  
    - Edge Cases: Invalid JSON output, missing data for placeholders  

  - **OpenAI Chat Model1 & OpenAI Chat Model2**  
    - Type: Language Model nodes  
    - Configuration: Supports AI Agent and Structured Output Parser respectively, running GPT-4.1-mini for refined email generation and parsing  
    - Edge Cases: API issues or timeouts  

  - **Structured Output Parser**  
    - Type: Output parser node  
    - Configuration: Validates and auto-fixes JSON format of AI Agent output  
    - Edge Cases: Parsing failures, malformed AI output  

  - **Send a message (Gmail Node)**  
    - Type: Email node  
    - Configuration: Sends an email to a configured recipient (placeholder email) with the AI-generated subject and body, including video and voice URLs.  
    - Input: Parsed JSON email content and video URL  
    - Edge Cases: SMTP auth errors, invalid email address  

---

#### 1.6 Polling & Control Flow

- **Overview:** Controls timing and progression between video generation, availability check, and email generation to ensure resources are ready before sending outreach.  
- **Nodes Involved:** Wait 30s, GET Result  
- **Node Details:**

  - Used to pause workflow to allow HeyGen video processing and then poll for video readiness using GET Result node before proceeding to email generation.  
  - Important for preventing premature email sending without video asset available.  
  - Edge Cases: Video processing delays longer than wait time, requiring multiple polls (not implemented here), potential failure if video never becomes ready.  

---

### 3. Summary Table

| Node Name                 | Node Type                                      | Functional Role                       | Input Node(s)              | Output Node(s)                          | Sticky Note                                                                                         |
|---------------------------|------------------------------------------------|-------------------------------------|----------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------|
| Google Sheets Trigger      | Trigger (Google Sheets Trigger)                 | Detect new lead row in sheet        | N/A                        | Code in JavaScript                     |                                                                                                   |
| Code in JavaScript        | Function                                        | Extract latest row from new data    | Google Sheets Trigger       | Research Agent                        | Sticky Note2: Research & Script                                                                    |
| Research Agent            | LangChain AI Agent                              | Perform AI research on lead/company | Code in JavaScript          | Scripting Agent                      | Sticky Note2: Research & Script                                                                    |
| Message a model in Perplexity | AI Tool (Perplexity)                         | Provide AI research data            | Research Agent              | Research Agent (feedback)             |                                                                                                   |
| Scripting Agent           | LangChain AI Agent                              | Write personalized outreach script | Research Agent              | Heygen Clone AI Creation, Convert text to speech | Sticky Note2: Research & Script                                                                    |
| OpenAI Chat Model         | Language Model (OpenAI GPT-5.1)                 | Support Research Agent              | N/A                        | Research Agent (ai_languageModel)     |                                                                                                   |
| OpenRouter Chat Model     | Language Model (OpenRouter GPT-5.1)             | Support Scripting Agent             | N/A                        | Scripting Agent (ai_languageModel)    |                                                                                                   |
| Heygen Clone AI Creation  | HTTP Request                                   | Generate personalized avatar video | Scripting Agent             | Wait 30s                             | Sticky Note: Intro Video Generation + Send to Email                                               |
| Wait 30s                  | Wait                                           | Delay for video processing          | Heygen Clone AI Creation    | GET Result                          | Sticky Note: Intro Video Generation + Send to Email                                               |
| GET Result                | HTTP Request                                   | Poll for video readiness            | Wait 30s                   | AI Agent                            | Sticky Note: Intro Video Generation + Send to Email                                               |
| AI Agent                  | LangChain AI Agent                              | Generate personalized outreach email| GET Result                 | Send a message                     | Sticky Note: Intro Video Generation + Send to Email                                               |
| OpenAI Chat Model1        | Language Model (OpenAI GPT-4.1-mini)             | Support AI Agent for email          | N/A                        | AI Agent (ai_languageModel)           |                                                                                                   |
| Structured Output Parser  | Output Parser                                   | Validate JSON email output          | OpenAI Chat Model2          | AI Agent (ai_outputParser)             |                                                                                                   |
| OpenAI Chat Model2        | Language Model (OpenAI GPT-4.1-mini)             | Support Structured Output Parser    | N/A                        | Structured Output Parser (ai_languageModel) |                                                                                                   |
| Send a message            | Gmail Node                                     | Send outreach email                 | AI Agent                   | N/A                                | Sticky Note: Intro Video Generation + Send to Email                                               |
| Convert text to speech    | ElevenLabs AI node                              | Generate voice note audio           | Scripting Agent             | Upload file                        | Sticky Note1: Voice Generation and Text                                                           |
| Upload file               | Google Drive node                              | Upload audio file to Drive          | Convert text to speech      | Send an SMS/MMS/WhatsApp message      | Sticky Note1: Voice Generation and Text                                                           |
| Send an SMS/MMS/WhatsApp message | Twilio node                             | Send SMS/WhatsApp with audio link  | Upload file                | N/A                                | Sticky Note1: Voice Generation and Text                                                           |
| Sticky Note               | Sticky Note                                    | Various informational notes         | N/A                        | N/A                                | See above notes for content                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node:**  
   - Type: Google Sheets Trigger  
   - Configure to poll every minute for new rows on your target Google Sheet and tab.  
   - Provide Google Sheets credentials and replace placeholders with your document ID and sheet tab name.  

2. **Add Code in JavaScript Node:**  
   - Connect from Google Sheets Trigger.  
   - Paste code to select the last item from incoming rows array.  
   - This extracts the latest lead for processing.  

3. **Add Research Agent Node (LangChain Agent):**  
   - Connect from Code in JavaScript node.  
   - Configure as LangChain Agent with Perplexity tool enabled.  
   - Set input text template to inject lead details (Full Name, Email, Phone, Company, Industry, LinkedIn URL).  
   - Use system prompt to instruct research on person, company, opportunities, and sales triggers with strict output format.  

4. **Add Message a model in Perplexity Node:**  
   - Connect as AI tool input to Research Agent node.  
   - Use "sonar-pro" model for researching the lead.  
   - No special parameters required beyond defaults.  

5. **Add Scripting Agent Node (LangChain Agent):**  
   - Connect from Research Agent node.  
   - Configure to receive research output and write a 30-second personalized outreach script.  
   - Use system prompt specifying tone, style, format (no jargon, casual, friendly, no filler, personalized references).  

6. **Add OpenAI Chat Model and OpenRouter Chat Model Nodes:**  
   - Connect OpenAI Chat Model as AI language model for Research Agent.  
   - Connect OpenRouter Chat Model as AI language model for Scripting Agent.  
   - Use latest GPT models available (e.g., GPT-5.1 for OpenAI, openai/gpt-5.1 for OpenRouter).  
   - Configure with appropriate API credentials.  

7. **Add HeyGen Clone AI Creation Node (HTTP Request):**  
   - Connect from Scripting Agent.  
   - Configure POST request to HeyGen API endpoint for video generation.  
   - Set JSON body to include avatar ID, voice ID, script text, and video dimensions.  
   - Use HTTP Header Authentication with HeyGen API key.  

8. **Add Wait Node (30s):**  
   - Connect from HeyGen Clone AI Creation node.  
   - Set to wait 30 seconds to allow video processing.  

9. **Add GET Result Node (HTTP Request):**  
   - Connect from Wait node.  
   - Configure GET request to HeyGen API video status endpoint with video_id query parameter from previous node output.  
   - Use same authentication as video generation node.  

10. **Add AI Agent Node (LangChain Agent) for Email:**  
    - Connect from GET Result node.  
    - Configure system prompt to generate a short personalized outreach email referencing the video URL.  
    - Require strict JSON output with "title" and "body" properties only.  

11. **Add OpenAI Chat Model1 and OpenAI Chat Model2 Nodes:**  
    - Connect to AI Agent to support language generation and output parsing.  
    - Use GPT-4.1-mini or similar models.  

12. **Add Structured Output Parser Node:**  
    - Connect from OpenAI Chat Model2 node.  
    - Configure JSON schema example to validate AI Agent's email output.  

13. **Add Send a message Node (Gmail):**  
    - Connect from AI Agent.  
    - Configure to send email to a designated email address.  
    - Use the JSON email subject and body from AI Agent output.  
    - Provide Gmail OAuth2 credentials.  

14. **Add Convert text to speech Node (ElevenLabs):**  
    - Connect from Scripting Agent node (parallel to HeyGen video generation).  
    - Configure ElevenLabs node with voice ID and API credentials to generate speech from the script text.  

15. **Add Upload file Node (Google Drive):**  
    - Connect from ElevenLabs node.  
    - Configure to upload the audio file to a specified Google Drive folder (replace folder ID).  
    - Provide Google Drive OAuth2 credentials.  

16. **Add Send an SMS/MMS/WhatsApp message Node (Twilio) [Optional]:**  
    - Connect from Upload file node.  
    - Configure with Twilio credentials and specify "to" and "from" phone numbers.  
    - Message body should include the audio file link URL from Google Drive.  

17. **Add Sticky Notes:**  
    - Add sticky notes for documentation and clarity at appropriate workflow locations describing block purposes and tutorial link.  

**Credentials Required:**  
- Google Sheets OAuth2  
- HeyGen API key (generic HTTP header auth)  
- ElevenLabs API key  
- Google Drive OAuth2  
- Gmail OAuth2  
- Twilio API credentials (optional)  

---

### 5. General Notes & Resources

| Note Content                                                                                                                      | Context or Link                                              |
|-----------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| Automates creation of hyper-personalized outreach videos, voice notes, and emails triggered by new Google Sheets leads.           | Workflow high-level purpose                                  |
| Watch step-by-step tutorial here: https://www.youtube.com/watch?v=q9AAh9zRou4                                                     | Video tutorial for this workflow                              |
| Uses multiple AI services: Perplexity for research, LangChain agents for AI orchestration, HeyGen for video generation, ElevenLabs for voice synthesis. | AI services used                                              |
| Workflow includes optional SMS/WhatsApp outreach via Twilio for voice note delivery.                                              | Optional communication channel                                |
| Replace all placeholder values (Google Sheet ID, Tab Name, Folder ID, phone numbers, API keys) before activating workflow.        | Critical setup instruction                                   |
| AI prompt designs enforce strict output formats and data integrity to minimize errors in downstream processing.                   | Prompt design best practices                                  |
| This workflow is designed for SDRs and marketing teams focusing on influencer marketing outreach personalization at scale.         | Target user base                                              |

---

**Disclaimer:**  
The provided text is exclusively generated from an n8n automated workflow. It strictly respects current content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly available.