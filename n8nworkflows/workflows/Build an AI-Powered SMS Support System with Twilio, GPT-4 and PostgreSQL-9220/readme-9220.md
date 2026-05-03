Build an AI-Powered SMS Support System with Twilio, GPT-4 and PostgreSQL

https://n8nworkflows.xyz/workflows/build-an-ai-powered-sms-support-system-with-twilio--gpt-4-and-postgresql-9220


# Build an AI-Powered SMS Support System with Twilio, GPT-4 and PostgreSQL

### 1. Workflow Overview

This workflow implements an AI-powered SMS support system integrating Twilio for SMS messaging, OpenAI GPT-4/Groq API for natural language understanding, and PostgreSQL for session and verification data management. It is designed to:

- Handle user signup and phone number verification via SMS.
- Process incoming SMS messages in a conversational AI flow.
- Analyze message sentiment, intent, urgency, and escalate critical cases.
- Store and enrich conversation data for context-aware responses.
- Generate AI-powered replies and personalized follow-ups.
- Close out sessions with summary transcripts and personalized thank you messages.

The workflow is logically divided into the following blocks:

- **1.1 User Signup & Verification:** Handles new user phone number registration, generates and sends verification codes, and validates user replies to establish verified sessions.
- **1.2 Incoming SMS Reception & Session Lookup:** Receives incoming SMS via webhook, extracts message data, and finds the corresponding user session.
- **1.3 AI-Based Sentiment, Intent, & Urgency Analysis:** Uses AI nodes to analyze incoming messages, extracting sentiment, intent, urgency, and keywords.
- **1.4 Escalation Handling:** Routes messages flagged as urgent or frustrated to escalation processes, notifying support via email and SMS, and marking sessions as escalated.
- **1.5 AI-Powered Conversational Flow:** Enriches session intake data with structured AI extraction, generates contextual AI responses, and sends replies via SMS.
- **1.6 Session Closeout:** Marks sessions as completed, uploads transcripts to S3, and sends personalized thank you SMS.
- **1.7 Follow-up Reminders:** Periodically finds inactive sessions and sends personalized AI-generated reminder SMS to re-engage users.

Sticky notes within the workflow provide guidance and context for each block.

---

### 2. Block-by-Block Analysis

#### 1.1 User Signup & Verification

**Overview:**  
This block manages phone number verification during user signup. It generates a code, sends it via Twilio SMS, waits for user response, validates the code, and creates an active session upon successful verification.

**Nodes Involved:**  
- User Signup Webhook1  
- Generate Verification Code1  
- Store Verification  
- Send Verification SMS (Twilio API)  
- Signup Response1  
- Verify Code Webhook1  
- Extract Verification Data1  
- Check Verification  
- Validate Code  
- Create Session  
- Mark Verified  
- Insert Session  
- Verify Success Response  
- Verify Failed Response  

**Node Details:**

- **User Signup Webhook1**  
  - Type: Webhook  
  - Role: Receives signup POST requests with user phone number.  
  - Config: HTTP POST on path `/user_signup`. Response mode returns JSON success with verification status.  
  - Inputs: External HTTP request.  
  - Outputs: Passes phone number to code generation.  
  - Edge Cases: Missing phone number, invalid format.

- **Generate Verification Code1**  
  - Type: Code  
  - Role: Generates random 6-digit verification code with 10-minute expiry.  
  - Config: JavaScript generates code, timestamps, and returns structured verification data.  
  - Inputs: Phone number from webhook.  
  - Outputs: Verification code + metadata for DB storage and SMS sending.  
  - Edge Cases: Random generation failure (rare).

- **Store Verification**  
  - Type: PostgreSQL Insert  
  - Role: Stores verification record in `user_verifications` table.  
  - Config: Insert operation with generated code and expiry.  
  - Inputs: Verification data.  
  - Outputs: Passes to Twilio SMS sender.  
  - Edge Cases: DB connection errors, write failures.

- **Send Verification SMS (Twilio API)**  
  - Type: HTTP Request  
  - Role: Sends SMS with verification code using Twilio REST API.  
  - Config: POST to Twilio Messages endpoint with HTTP Basic Auth (configured in credentials).  
  - Parameters: To = user phone, From = Twilio number, Body = verification message.  
  - Inputs: Verification record from DB insert.  
  - Outputs: Confirmation for webhook response.  
  - Edge Cases: Twilio API errors, invalid phone numbers.

- **Signup Response1**  
  - Type: Respond to Webhook  
  - Role: Returns JSON success response confirming SMS sent.  
  - Inputs: Result from SMS sending node.  
  - Outputs: HTTP response to initial signup request.

- **Verify Code Webhook1**  
  - Type: Webhook  
  - Role: Receives user POST with phone number and verification code for validation.  
  - Config: HTTP POST on path `/verify_code`.  
  - Inputs: External HTTP request.  
  - Outputs: Passes data for extraction and validation.

- **Extract Verification Data1**  
  - Type: Code  
  - Role: Extracts phone number, submitted code, and current time for validation.  
  - Inputs: Webhook data.  
  - Outputs: Structured info for DB query.

- **Check Verification**  
  - Type: PostgreSQL Select  
  - Role: Queries `user_verifications` for matching phone number, unverified, latest record.  
  - Inputs: Extracted verification data.  
  - Outputs: Verification record for code comparison.

- **Validate Code**  
  - Type: Switch  
  - Role: Compares submitted code to stored code and expiry; branches workflow.  
  - Inputs: Verification record and submitted code.  
  - Outputs: Success branch to Create Session; failure branch to Verify Failed Response.  
  - Edge Cases: Expired or incorrect codes.

- **Create Session**  
  - Type: Code  
  - Role: Creates new user session with unique session ID, empty intake data, and active status.  
  - Inputs: Verified phone number from verification data.  
  - Outputs: Session data for DB insertion.

- **Mark Verified**  
  - Type: PostgreSQL Update  
  - Role: Marks verification record as verified = true.  
  - Inputs: Phone number and verification record.  
  - Outputs: Confirmation.

- **Insert Session**  
  - Type: PostgreSQL Insert  
  - Role: Inserts new session record into `user_sessions` table.  
  - Inputs: Session data from Create Session.  
  - Outputs: Confirmation.

- **Verify Success Response**  
  - Type: Respond to Webhook  
  - Role: Returns JSON success with session ID.  
  - Inputs: Confirmation from session insert.  
  - Outputs: HTTP response to verification request.

- **Verify Failed Response**  
  - Type: Respond to Webhook  
  - Role: Returns JSON failure message for invalid or expired codes.  
  - Inputs: Failed validation branch.  
  - Outputs: HTTP response.

---

#### 1.2 Incoming SMS Reception & Session Lookup

**Overview:**  
Receives incoming SMS messages from Twilio webhook, extracts message data, and finds the active user session matching the sender’s phone number.

**Nodes Involved:**  
- Incoming SMS Webhook (Twilio)  
- Extract SMS Data / Extract SMS Data1 (two instances, functionally similar)  
- Find User Session / Find User Session1  

**Node Details:**

- **Incoming SMS Webhook / Incoming SMS Webhook (Twilio)**  
  - Type: Webhook  
  - Role: Entry point for inbound SMS POST requests from Twilio.  
  - Config: POST on `/twilio_sms_incoming`. Response mode set to respond after processing.  
  - Inputs: HTTP requests with SMS data.  
  - Outputs: Raw SMS JSON.

- **Extract SMS Data / Extract SMS Data1**  
  - Type: Code  
  - Role: Extracts key SMS fields: sender phone number, message body, message SID, and timestamps the reception.  
  - Inputs: Webhook JSON.  
  - Outputs: Structured SMS data for session lookup and AI processing.  
  - Edge Cases: Missing fields in webhook payload.

- **Find User Session / Find User Session1**  
  - Type: PostgreSQL Select  
  - Role: Queries `user_sessions` table for the most recent active session matching the phone number.  
  - Config: Sort by `created_at DESC`, limit 1.  
  - Inputs: Extracted phone number from SMS.  
  - Outputs: Session data or empty if none found.  
  - Edge Cases: No active session found, database access errors.

---

#### 1.3 AI-Based Sentiment, Intent, & Urgency Analysis

**Overview:**  
Passes incoming SMS message body to AI models (Groq or OpenAI GPT-4) to analyze sentiment, intent, urgency, and keywords. Processes and enriches AI output for session context.

**Nodes Involved:**  
- AI Sentiment & Intent Analysis (Groq HTTP Request)  
- 🤖 AI Sentiment Analysis (OpenAI node)  
- Process & Store AI Analysis / Process AI Analysis (Code nodes)  

**Node Details:**

- **AI Sentiment & Intent Analysis**  
  - Type: HTTP Request (Groq API)  
  - Role: Sends SMS text to Groq API for detailed JSON response including sentiment, urgency score, intent, escalation flag, keywords, summary.  
  - Config: POST with chat completion payload, low temperature 0.3, max tokens 300.  
  - Inputs: Message body from Extract SMS Data.  
  - Outputs: Raw AI analysis JSON.

- **🤖 AI Sentiment Analysis**  
  - Type: OpenAI GPT-4  
  - Role: Alternative AI sentiment and intent analysis using OpenAI’s GPT-4 model.  
  - Config: GPT-4, max tokens 500, temperature 0.3.  
  - Inputs: Message body from Extract SMS Data1.  
  - Outputs: AI JSON analysis string.

- **Process & Store AI Analysis / Process AI Analysis**  
  - Type: Code  
  - Role: Parses AI JSON response, sanitizes it, and merges AI analysis with existing session intake data. Updates transcript and AI metadata fields.  
  - Inputs: AI raw JSON, session data from Find User Session* nodes, SMS data.  
  - Outputs: Updated session data with AI insights, ready for DB update or routing.  
  - Edge Cases: Invalid or malformed AI JSON responses, missing session data.

---

#### 1.4 Escalation Handling

**Overview:**  
Routes sessions flagged by AI as requiring escalation due to high urgency or frustration. Prepares escalation context, updates session status, sends alerts via email and SMS to support team, and sends supportive SMS to user.

**Nodes Involved:**  
- AI Escalation Router / 🤖 AI Escalation Router (Switch nodes)  
- Prepare Escalation Data / Prepare Escalation Data1 (Code nodes)  
- Mark Session as Escalated / Mark Session Escalated (Postgres update)  
- Send AI Escalation Email / Send Escalation Email (Gmail nodes)  
- Send Escalation SMS / Send Escalation SMS1 (Twilio HTTP Request)  

**Node Details:**

- **AI Escalation Router / 🤖 AI Escalation Router**  
  - Type: Switch  
  - Role: Evaluates AI analysis fields (e.g., requires_escalation, sentiment, urgency_score) to decide if escalation is needed.  
  - Inputs: Processed AI analysis node output.  
  - Outputs: Routes to escalation or normal handling paths.  
  - Edge Cases: Misclassification by AI, missing AI flags.

- **Prepare Escalation Data / Prepare Escalation Data1**  
  - Type: Code  
  - Role: Prepares a JSON object with session ID, phone number, escalation timestamp, AI analysis details for DB update.  
  - Inputs: AI analysis and session data.  
  - Outputs: Escalation update object.

- **Mark Session as Escalated / Mark Session Escalated**  
  - Type: PostgreSQL Update  
  - Role: Updates session status to `escalated` in `user_sessions` table.  
  - Inputs: Escalation data.  
  - Outputs: Confirmation.

- **Send AI Escalation Email / Send Escalation Email**  
  - Type: Gmail  
  - Role: Sends email alert to support team with AI analysis summary and session info.  
  - Config: Sends to `support@company.com` with dynamic subject and detailed message body.  
  - Inputs: Escalation session data.  
  - Outputs: Confirmation.

- **Send Escalation SMS / Send Escalation SMS1**  
  - Type: HTTP Request (Twilio)  
  - Role: Sends an SMS to the user indicating the issue was escalated and support will contact them soon.  
  - Inputs: Session phone number and ID.  
  - Outputs: Confirmation.

---

#### 1.5 AI-Powered Conversational Flow

**Overview:**  
Based on user intent extracted by AI, this block enriches session intake data with structured information, generates AI-crafted SMS responses, updates session data in the database, and sends SMS replies.

**Nodes Involved:**  
- 🤖 AI Intent Router (Switch)  
- 🤖 AI Intake Enrichment (OpenAI GPT-4)  
- Merge Enriched Data (Code)  
- Save Enriched Intake1 (Postgres Update)  
- 🤖 AI Response Generator (OpenAI GPT-4)  
- Send AI Response SMS (Twilio HTTP Request)  

**Node Details:**

- **🤖 AI Intent Router**  
  - Type: Switch  
  - Role: Routes flow based on AI-detected intent: continue intake, complete session, help request, etc.  
  - Inputs: Process AI Analysis node output.  
  - Outputs: Branches for different intents triggering enrichment or closeout.

- **🤖 AI Intake Enrichment**  
  - Type: OpenAI GPT-4  
  - Role: Extracts structured data fields, completeness score, suggested next question, and category from free-text input.  
  - Inputs: User message content.  
  - Outputs: JSON with extracted info and meta.

- **Merge Enriched Data**  
  - Type: Code  
  - Role: Merges new extracted info into existing intake data’s structured_data, updates AI metadata with completeness and suggested question.  
  - Inputs: AI enrichment output and session intake data.  
  - Outputs: Updated intake data for DB update.

- **Save Enriched Intake1**  
  - Type: PostgreSQL Update  
  - Role: Saves updated intake data JSON back into `user_sessions`.  
  - Inputs: Merged intake data.  
  - Outputs: Confirmation.

- **🤖 AI Response Generator**  
  - Type: OpenAI GPT-4  
  - Role: Generates a context-aware SMS reply message based on current conversation and AI analysis.  
  - Inputs: Enriched intake data and message context.  
  - Outputs: SMS message content.

- **Send AI Response SMS**  
  - Type: HTTP Request (Twilio)  
  - Role: Sends AI-generated SMS reply to user via Twilio API.  
  - Inputs: Phone number and AI message content.  
  - Outputs: Confirmation.

---

#### 1.6 Session Closeout

**Overview:**  
Handles session completion by marking the session as closed, uploading conversation transcript to AWS S3, generating a personalized thank-you SMS, and sending it to the user.

**Nodes Involved:**  
- Prepare Closeout (Code)  
- Finalize Session (Postgres Update)  
- Upload to S3 (AWS S3)  
- 🤖 AI Personalized Closeout (OpenAI GPT-4)  
- Send Closeout SMS (Twilio HTTP Request)  

**Node Details:**

- **Prepare Closeout**  
  - Type: Code  
  - Role: Marks intake data as completed with timestamp, generates filename for transcript upload.  
  - Inputs: Current intake data and session info.  
  - Outputs: Updated session data including final transcript string and S3 path.

- **Finalize Session**  
  - Type: PostgreSQL Update  
  - Role: Updates session status to `completed`, stores final transcript and completion timestamp.  
  - Inputs: Closeout data from Prepare Closeout.  
  - Outputs: Confirmation.

- **Upload to S3**  
  - Type: AWS S3  
  - Role: Uploads JSON transcript file to S3 bucket with generated filename.  
  - Inputs: Transcript JSON and filename.  
  - Outputs: Confirmation.

- **🤖 AI Personalized Closeout**  
  - Type: OpenAI GPT-4  
  - Role: Generates a personalized thank-you message for the user at session end.  
  - Inputs: Session summary and intake data.  
  - Outputs: SMS message content.

- **Send Closeout SMS**  
  - Type: HTTP Request (Twilio)  
  - Role: Sends personalized thank-you SMS via Twilio.  
  - Inputs: User phone number and AI message content.  
  - Outputs: Confirmation.

---

#### 1.7 Follow-up Reminders

**Overview:**  
Periodically runs a cron job to identify inactive sessions (no updates for 24 hours), generates personalized AI reminder messages, and sends SMS reminders to re-engage users.

**Nodes Involved:**  
- Follow-Up Cron (24h)  
- Find Inactive Sessions (Postgres Select)  
- 🤖 AI Personalized Reminder (OpenAI GPT-4)  
- Send Reminder SMS (Twilio HTTP Request)  

**Node Details:**

- **Follow-Up Cron (24h)**  
  - Type: Cron  
  - Role: Triggers workflow every 24 hours.  
  - Inputs: None.  
  - Outputs: Starts query for inactive sessions.

- **Find Inactive Sessions**  
  - Type: PostgreSQL Select  
  - Role: Selects sessions with `status = active` and `updated_at` older than 24 hours.  
  - Outputs: List of sessions for reminder.

- **🤖 AI Personalized Reminder**  
  - Type: OpenAI GPT-4  
  - Role: Generates contextual reminder SMS content based on session data.  
  - Inputs: Session data from DB.  
  - Outputs: Reminder message content.

- **Send Reminder SMS**  
  - Type: HTTP Request (Twilio)  
  - Role: Sends reminder SMS to user via Twilio API.  
  - Inputs: Phone number and AI-generated message.  
  - Outputs: Confirmation.

---

### 3. Summary Table

| Node Name                     | Node Type           | Functional Role                           | Input Node(s)                     | Output Node(s)                  | Sticky Note                                                                                  |
|-------------------------------|---------------------|-----------------------------------------|----------------------------------|--------------------------------|----------------------------------------------------------------------------------------------|
| User Signup Webhook1           | Webhook             | Receives user signup requests           |                                  | Generate Verification Code1     | ## User Signup                                                                              |
| Generate Verification Code1    | Code                | Generates 6-digit verification code     | User Signup Webhook1             | Store Verification             | ## User Signup                                                                              |
| Store Verification            | PostgreSQL Insert    | Stores verification record               | Generate Verification Code1      | Send Verification SMS (Twilio API) | ## User Signup                                                                              |
| Send Verification SMS (Twilio API) | HTTP Request       | Sends verification SMS via Twilio       | Store Verification              | Signup Response1               | ## User Signup                                                                              |
| Signup Response1              | Respond to Webhook   | Sends signup success response            | Send Verification SMS (Twilio API) |                                | ## User Signup                                                                              |
| Verify Code Webhook1          | Webhook             | Receives verification code POST          |                                  | Extract Verification Data1     | ## Verification Agent                                                                       |
| Extract Verification Data1    | Code                | Extracts phone and code from POST        | Verify Code Webhook1             | Check Verification             | ## Verification Agent                                                                       |
| Check Verification            | PostgreSQL Select    | Checks latest unverified code            | Extract Verification Data1       | Validate Code                  | ## Verification Agent                                                                       |
| Validate Code                | Switch              | Validates code, routes success/failure  | Check Verification              | Create Session, Verify Failed Response | ## Verification Agent                                                                       |
| Create Session               | Code                | Creates new user session                  | Validate Code (success branch)  | Mark Verified, Insert Session  | ## Verification Agent                                                                       |
| Mark Verified                | PostgreSQL Update    | Marks verification record as verified    | Create Session                  | Verify Success Response        | ## Verification Agent                                                                       |
| Insert Session               | PostgreSQL Insert    | Inserts new active session                | Create Session                  | Verify Success Response        | ## Verification Agent                                                                       |
| Verify Success Response      | Respond to Webhook   | Sends verification success response      | Mark Verified, Insert Session   |                                | ## Verification Agent                                                                       |
| Verify Failed Response       | Respond to Webhook   | Sends verification failure response      | Validate Code (failure branch)  |                                | ## Verification Agent                                                                       |
| Incoming SMS Webhook (Twilio) | Webhook             | Receives incoming SMS messages            |                                  | Extract SMS Data1              | ## Escalation Agent                                                                         |
| Extract SMS Data / Extract SMS Data1 | Code                | Extracts SMS payload data                 | Incoming SMS Webhook             | Find User Session / Find User Session1 | ## Escalation Agent                                                                         |
| Find User Session / Find User Session1 | PostgreSQL Select    | Finds latest active user session          | Extract SMS Data / Extract SMS Data1 | AI Sentiment & Intent Analysis / 🤖 AI Sentiment Analysis | ## Escalation Agent                                                                         |
| AI Sentiment & Intent Analysis | HTTP Request (Groq)  | AI analyzes sentiment, intent, urgency  | Find User Session               | Process & Store AI Analysis    | Placeholder: Swap to AI node after import                                                  |
| 🤖 AI Sentiment Analysis      | OpenAI GPT-4         | AI analyzes sentiment, intent, urgency  | Find User Session1              | Process AI Analysis            | AI NODE 1: Analyzes sentiment, urgency, intent from free-text                              |
| Process & Store AI Analysis / Process AI Analysis | Code                | Parses AI results, updates session data  | AI Sentiment & Intent Analysis / 🤖 AI Sentiment Analysis | AI Escalation Router / 🤖 AI Escalation Router   | Processes AI output and stores it with transcript in structured format                      |
| AI Escalation Router / 🤖 AI Escalation Router | Switch              | Routes escalation based on AI flags      | Process & Store AI Analysis / Process AI Analysis | Prepare Escalation Data / Prepare Escalation Data1, 🤖 AI Intent Router | AI NODE 2: Routes to escalation based on AI-detected frustration/urgency                   |
| Prepare Escalation Data / Prepare Escalation Data1 | Code                | Prepares escalation session update data  | AI Escalation Router / 🤖 AI Escalation Router | Mark Session as Escalated / Mark Session Escalated |                                                                                              |
| Mark Session as Escalated / Mark Session Escalated | PostgreSQL Update    | Updates session status to escalated       | Prepare Escalation Data / Prepare Escalation Data1 | Send AI Escalation Email / Send Escalation Email |                                                                                              |
| Send AI Escalation Email / Send Escalation Email | Gmail                | Sends escalation email alert to support   | Mark Session as Escalated / Mark Session Escalated | Send Escalation SMS / Send Escalation SMS1 |                                                                                              |
| Send Escalation SMS / Send Escalation SMS1 | HTTP Request (Twilio) | Sends escalation SMS notification to user | Send AI Escalation Email / Send Escalation Email | SMS Webhook Response           |                                                                                              |
| 🤖 AI Intent Router          | Switch              | Routes based on AI-detected user intent   | 🤖 AI Escalation Router         | Prepare Closeout, 🤖 AI Intake Enrichment | AI NODE 3: Routes based on AI-detected user intent                                        |
| 🤖 AI Intake Enrichment      | OpenAI GPT-4         | Extracts structured intake data            | 🤖 AI Intent Router             | Merge Enriched Data            | AI NODE 4: Extracts structured data from free-text                                        |
| Merge Enriched Data          | Code                | Merges AI extraction into intake data      | 🤖 AI Intake Enrichment         | Save Enriched Intake1          |                                                                                              |
| Save Enriched Intake1        | PostgreSQL Update    | Saves updated intake data to DB             | Merge Enriched Data             | 🤖 AI Response Generator       |                                                                                              |
| 🤖 AI Response Generator     | OpenAI GPT-4         | Generates AI SMS reply content              | Save Enriched Intake1           | Send AI Response SMS           | AI NODE 5: Generates contextual SMS responses                                             |
| Send AI Response SMS         | HTTP Request (Twilio) | Sends AI-generated SMS reply                 | 🤖 AI Response Generator        | SMS Webhook Response           |                                                                                              |
| Prepare Closeout             | Code                | Marks intake complete, generates transcript | 🤖 AI Intent Router             | Finalize Session, Upload to S3 |                                                                                              |
| Finalize Session            | PostgreSQL Update    | Updates session status to completed          | Prepare Closeout               | 🤖 AI Personalized Closeout    |                                                                                              |
| Upload to S3                | AWS S3               | Uploads transcript JSON file to S3 bucket    | Prepare Closeout               |                                |                                                                                              |
| 🤖 AI Personalized Closeout  | OpenAI GPT-4         | Generates personalized thank-you message      | Finalize Session               | Send Closeout SMS              | AI NODE 6: Generates personalized thank you                                               |
| Send Closeout SMS            | HTTP Request (Twilio) | Sends personalized thank-you SMS               | 🤖 AI Personalized Closeout    | SMS Webhook Response           |                                                                                              |
| SMS Webhook Response         | Respond to Webhook   | Sends "OK" text response to Twilio webhook   | Send AI Response SMS, Send Escalation SMS1, Send Closeout SMS |                                |                                                                                              |
| Follow-Up Cron (24h)         | Cron                 | Triggers follow-up reminder workflow daily    |                                | Find Inactive Sessions         | ## Followup agent                                                                           |
| Find Inactive Sessions       | PostgreSQL Select    | Finds active sessions inactive >24hrs          | Follow-Up Cron                 | 🤖 AI Personalized Reminder    |                                                                                              |
| 🤖 AI Personalized Reminder  | OpenAI GPT-4         | Generates AI reminder SMS content               | Find Inactive Sessions         | Send Reminder SMS              | AI NODE 7: Generates contextual follow-up reminders                                      |
| Send Reminder SMS            | HTTP Request (Twilio) | Sends reminder SMS via Twilio                    | 🤖 AI Personalized Reminder    |                                |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create User Signup Webhook**  
   - Type: Webhook  
   - HTTP Method: POST  
   - Path: `/user_signup`  
   - Response Mode: Respond after processing

2. **Add Code Node "Generate Verification Code"**  
   - Generate random 6-digit code, expiration 10 minutes from now  
   - Input: phone_number from webhook  
   - Output: verification_code, phone_number, expires_at, created_at, verified=false

3. **Add PostgreSQL Insert Node "Store Verification"**  
   - Table: `user_verifications`  
   - Insert verification record with above fields

4. **Add HTTP Request Node "Send Verification SMS (Twilio API)"**  
   - URL: `https://api.twilio.com/2010-04-01/Accounts/{{ $credentials.accountSid }}/Messages.json`  
   - Method: POST  
   - Auth: HTTP Basic (Twilio credentials)  
   - Body: To (user phone), From (Twilio number), Body (verification message)  

5. **Add Respond to Webhook Node "Signup Response"**  
   - Respond with JSON success message confirming SMS sent

6. **Connect nodes sequentially: User Signup Webhook → Generate Verification Code → Store Verification → Send Verification SMS → Signup Response**

7. **Create Webhook Node "Verify Code Webhook"**  
   - POST method, path `/verify_code`  
   - Response mode: respond after processing

8. **Add Code Node "Extract Verification Data"**  
   - Extract phone_number, submitted_code, current_time from webhook data

9. **Add PostgreSQL Select Node "Check Verification"**  
   - Table: `user_verifications`  
   - Conditions: phone_number matches, verified=false  
   - Sort: created_at DESC, limit 1

10. **Add Switch Node "Validate Code"**  
    - Condition: submitted_code matches stored code and not expired  
    - True path: Create Session  
    - False path: Verify Failed Response

11. **Add Code Node "Create Session"**  
    - Generate unique session_id  
    - Initialize empty intake_data with transcript and AI metadata  
    - Set status = active, created_at timestamp

12. **Add PostgreSQL Update Node "Mark Verified"**  
    - Update `user_verifications` verified = true

13. **Add PostgreSQL Insert Node "Insert Session"**  
    - Insert new session into `user_sessions`

14. **Add Respond to Webhook Nodes "Verify Success Response" and "Verify Failed Response"**  
    - Success: JSON with success and session_id  
    - Failure: JSON with failure message

15. **Connect Verify Code Webhook → Extract Verification Data → Check Verification → Validate Code → (success) Create Session → Mark Verified & Insert Session → Verify Success Response**  
    → (failure) Verify Failed Response

16. **Create Incoming SMS Webhook Node**  
    - POST method, path `/twilio_sms_incoming`  
    - Response mode: respond after processing

17. **Add Code Node "Extract SMS Data"**  
    - Extract From, Body, MessageSid, add ISO timestamp

18. **Add PostgreSQL Select Node "Find User Session"**  
    - Table: `user_sessions`  
    - Condition: phone_number equals extracted number  
    - Limit 1, order by created_at DESC

19. **Add AI HTTP Request Node "AI Sentiment & Intent Analysis"**  
    - POST to Groq or OpenAI API  
    - Send message body with prompt for JSON response with sentiment, intent, urgency, escalation flag, keywords, summary  
    - Use credentials for API access

20. **Add Code Node "Process & Store AI Analysis"**  
    - Parse AI response JSON  
    - Merge AI data with session intake_data transcript and AI metadata  
    - Update fields like last_sentiment, max_urgency, escalation_count

21. **Add Switch Node "AI Escalation Router"**  
    - Route based on requires_escalation or urgency threshold

22. **Add Code Node "Prepare Escalation Data"**  
    - Prepare escalation info for DB update

23. **Add PostgreSQL Update Node "Mark Session as Escalated"**

24. **Add Gmail Node "Send AI Escalation Email"**  
    - Send alert email with session and AI analysis details

25. **Add HTTP Request Node "Send Escalation SMS"**  
    - Send SMS to user notifying escalation

26. **Add Switch Node "AI Intent Router"**  
    - Route based on intent (continue intake, complete session, help request)

27. **Add OpenAI Node "AI Intake Enrichment"**  
    - Extract structured info, completeness, suggested next question

28. **Add Code Node "Merge Enriched Data"**  
    - Merge extracted info into intake_data.structured_data

29. **Add PostgreSQL Update Node "Save Enriched Intake"**

30. **Add OpenAI Node "AI Response Generator"**  
    - Generate SMS reply text

31. **Add HTTP Request Node "Send AI Response SMS"**

32. **Add Respond to Webhook Node "SMS Webhook Response"**  
    - Respond "OK" to Twilio webhook

33. **Add Code Node "Prepare Closeout"**  
    - Mark intake_data completed, create transcript filename

34. **Add PostgreSQL Update Node "Finalize Session"**

35. **Add AWS S3 Node "Upload to S3"**  
    - Upload transcript JSON file

36. **Add OpenAI Node "AI Personalized Closeout"**  
    - Generate thank-you message

37. **Add HTTP Request Node "Send Closeout SMS"**

38. **Add Cron Node "Follow-Up Cron (24h)"**

39. **Add PostgreSQL Select Node "Find Inactive Sessions"**  
    - Sessions active but no update for 24 hours

40. **Add OpenAI Node "AI Personalized Reminder"**  
    - Generate reminder SMS

41. **Add HTTP Request Node "Send Reminder SMS"**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                         | Context or Link                                                                                      |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow integrates Twilio SMS, OpenAI GPT-4 (or Groq API), PostgreSQL, and AWS S3 to provide AI-enhanced conversational support.                                                                                                                                                                                                                                   | Workflow description                                                                                |
| Setup prerequisites include Twilio account with phone number configured for webhook, OpenAI API key, PostgreSQL database (e.g., Supabase or Neon), and AWS credentials for S3 upload.                                                                                                                                                                                    | Setup instructions                                                                                  |
| Twilio webhook URL must be configured in Twilio console to point to the n8n webhook path `/twilio_sms_incoming`.                                                                                                                                                                                                                                                     | Twilio setup                                                                                       |
| AI nodes require valid API credentials stored in n8n credentials manager (OpenAI API key, Groq API key).                                                                                                                                                                                                                                                             | Credentials setup                                                                                   |
| The workflow uses a combination of OpenAI GPT-4 and Groq APIs for AI processing; placeholders indicate where to swap nodes depending on your setup.                                                                                                                                                                                                                 | AI node configuration                                                                              |
| Sticky notes within the workflow provide modular grouping and guidance: User Signup, Verification Agent, Escalation Agent, Response Agent, Help Request, Followup Agent, Close Out, Human Escalation.                                                                                                                                                                  | Workflow sticky notes                                                                               |
| The workflow supports extensibility for human agents to intervene on escalated sessions, and can be adapted to other conversational AI scenarios beyond SMS.                                                                                                                                                                                                       | Extensibility note                                                                                  |
| For troubleshooting: monitor Twilio webhook logs, PostgreSQL query logs, and API request responses for AI and email nodes. Handle edge cases such as message parsing errors, AI response failures, and expired verification codes gracefully with fallback logic.                                                                                                     | Operational tips                                                                                   |
| Related blog post and setup video available at: https://n8n.io/blog/build-ai-powered-sms-support-system-twilio-gpt-postgresql-n8n                                                                                                                                                                                                                                   | External resource                                                                                   |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. It adheres strictly to content policies and contains no illegal or offensive elements. All data processed is legal and public.