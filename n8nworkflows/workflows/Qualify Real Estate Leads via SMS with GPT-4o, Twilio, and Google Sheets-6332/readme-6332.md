Qualify Real Estate Leads via SMS with GPT-4o, Twilio, and Google Sheets

https://n8nworkflows.xyz/workflows/qualify-real-estate-leads-via-sms-with-gpt-4o--twilio--and-google-sheets-6332


# Qualify Real Estate Leads via SMS with GPT-4o, Twilio, and Google Sheets

### 1. Workflow Overview

This workflow automates the qualification of real estate leads submitted via a web form by engaging with them through SMS using Twilio and GPT-4o AI models. It captures leads’ phone numbers, initiates a friendly AI-driven conversation to gather key buyer information, stores the conversation context in a PostgreSQL database (Supabase), and finally summarizes and logs the lead details into Google Sheets while notifying the agent via SMS.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception and Initial Contact:** Capture lead phone number via form submission, wait briefly, then send an initial SMS message inviting the lead to answer qualification questions.
- **1.2 Lead Conversation Handling:** Listen for inbound SMS replies, run a GPT-4o-powered AI agent to ask and process three qualification questions, and maintain chat memory for context.
- **1.3 Conversation Completion Check & Responses:** Detect when the conversation is complete, respond with a thank you message, or continue the AI conversation flow.
- **1.4 Conversation Summarization and Lead Logging:** Query conversation history, summarize the transcript and key points with AI, store results in Google Sheets, and notify the realtor with a summary SMS.
- **1.5 Supportive Utility Nodes:** Wait nodes to control timing, output parsing, and sticky notes for documentation and instructions.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initial Contact

- **Overview:**  
  This block captures the phone number from a form submission and sends an initial SMS to start the qualification conversation.

- **Nodes Involved:**  
  - On form submission  
  - Wait 5 Seconds  
  - Initial Text Message  
  - Sticky Note3 (Documentation)  

- **Node Details:**

  - **On form submission**  
    - Type: Form Trigger  
    - Role: Entry point triggered by submission of the "Real Estate Inquiry" form with a required "Phone" field.  
    - Config: Listens for form submissions; expects phone number with country code (e.g., +1).  
    - Inputs: Webhook triggered by form submit  
    - Outputs: Passes form data downstream  
    - Edge Cases: Missing/invalid phone number, webhook downtime  
    - Notes: Ensures phone number is correctly formatted to avoid Twilio send errors

  - **Wait 5 Seconds**  
    - Type: Wait  
    - Role: Adds a 10-second delay after form submission before sending SMS (possibly to ensure systems are ready).  
    - Config: Wait time set to 10 seconds.  
    - Inputs: From form submission node  
    - Outputs: Triggers next node after delay  
    - Edge Cases: Delay longer than needed may slow workflow

  - **Initial Text Message**  
    - Type: Twilio Node  
    - Role: Sends initial SMS to the phone number captured, inviting the lead to answer questions.  
    - Config:  
      - To: Phone from form submission (`{{$json.Phone}}`)  
      - From: Twilio number `+17179155475`  
      - Message: "Thanks for submitting the form on our site realestate.com. Do you have a few minutes to answer a few questions?"  
    - Inputs: After wait node  
    - Outputs: SMS sent confirmation  
    - Credentials: Twilio API credentials required  
    - Edge Cases: SMS send failure (invalid number, Twilio limits)  
    - Notes: Twilio phone number must have SMS enabled

  - **Sticky Note3**  
    - Role: Documentation for this block  
    - Content: "## Initial Text Message after Form Submission"

#### 2.2 Lead Conversation Handling

- **Overview:**  
  This block listens for inbound SMS replies and uses an AI-driven agent to ask three qualification questions, maintaining chat memory in a PostgreSQL database.

- **Nodes Involved:**  
  - Wait for Text Response  
  - Wait  
  - Real Estate Qualifier (AI Agent)  
  - Postgres Chat Memory  
  - OpenAI Chat Model2  
  - Sticky Note4 (Documentation)  

- **Node Details:**

  - **Wait for Text Response**  
    - Type: Twilio Trigger  
    - Role: Listens for incoming SMS messages to the Twilio number.  
    - Config: Listens to inbound message events (`com.twilio.messaging.inbound-message.received`).  
    - Inputs: Webhook triggered by incoming SMS  
    - Outputs: Passes SMS data downstream  
    - Credentials: Twilio API credentials  
    - Edge Cases: Twilio webhook misconfiguration, message delays

  - **Wait**  
    - Type: Wait  
    - Role: Adds a delay before processing the incoming message to handle any timing issues gracefully.  
    - Config: No specific parameters (default wait)  
    - Inputs: Inbound SMS from Twilio Trigger  
    - Outputs: Triggers AI agent node

  - **Real Estate Qualifier**  
    - Type: LangChain AI Agent  
    - Role: AI-driven conversational agent that qualifies the lead by asking three brief questions and ends politely.  
    - Config:  
      - Input text: Incoming SMS body (`{{$json.data.body}}`)  
      - System message prompt instructs agent to:  
        1. Ask for name  
        2. Ask type of property (house, condo, rental)  
        3. Ask city or neighborhood  
        - End conversation politely after all three answered, appending "***" to signal completion  
    - Inputs: Incoming SMS text, chat memory from Postgres  
    - Outputs: AI-generated response text  
    - Credentials: OpenAI API (GPT-4o), PostgreSQL Chat Memory for session context (session key = sender phone number)  
    - Edge Cases: AI response failures, OpenAI API rate limits, memory retrieval errors  
    - Notes: Uses session key based on phone number to keep context per lead

  - **Postgres Chat Memory**  
    - Type: LangChain Memory Node (PostgreSQL)  
    - Role: Stores and retrieves chat history to maintain context across messages  
    - Config: Uses session key derived from incoming phone number (`{{ $('Twilio Trigger').item.json.data.from }}`)  
    - Inputs: Connected to AI agent node  
    - Outputs: Provides chat memory to AI agent  
    - Credentials: PostgreSQL connection linked to Supabase instance  
    - Edge Cases: Database connectivity issues, session key collisions

  - **OpenAI Chat Model2**  
    - Type: LangChain OpenAI Chat Model  
    - Role: Provides GPT-4o model for AI agent to generate messages  
    - Config: Model set to `gpt-4o`  
    - Inputs: From AI Agent node  
    - Outputs: AI-generated chat response  
    - Credentials: OpenAI API key  
    - Edge Cases: API latency or downtime, token limits

  - **Sticky Note4**  
    - Role: Documentation for this block  
    - Content: "## Wait for text response & Reply"

#### 2.3 Conversation Completion Check & Responses

- **Overview:**  
  This block determines if the AI conversation has concluded (detected by "***"), sends a thank you SMS if complete, or continues the conversation by sending AI-generated replies.

- **Nodes Involved:**  
  - Check if Conversation Completed (If node)  
  - Thank Your Text  
  - Response Text from Agent  
  - Sticky Note5 (Documentation)  

- **Node Details:**

  - **Check if Conversation Completed**  
    - Type: If  
    - Role: Checks if AI response output contains the completion marker "***" indicating conversation end.  
    - Config: Condition: `{{$json.output}}` contains `"***"` string (case sensitive).  
    - Inputs: From Real Estate Qualifier node (AI output)  
    - Outputs:  
      - True (conversation complete): triggers Thank Your Text and Query Supabase for conversation history  
      - False (conversation ongoing): triggers Response Text from Agent node  
    - Edge Cases: Missing "***" marker due to AI output variations, false positives

  - **Thank Your Text**  
    - Type: Twilio Node  
    - Role: Sends a polite thank you SMS to the lead when conversation finishes.  
    - Config:  
      - To: inbound phone number (`{{ $('Wait for Text Response').item.json.data.from }}`)  
      - From: Twilio number  
      - Message: "Thanks, I'll give you a call soon."  
    - Credentials: Twilio API  
    - Edge Cases: SMS delivery failure

  - **Response Text from Agent**  
    - Type: Twilio Node  
    - Role: Sends the AI agent’s reply to the lead to continue the conversation.  
    - Config:  
      - To: inbound phone number  
      - From: Twilio number  
      - Message: AI output text (`{{$json.output}}`)  
    - Credentials: Twilio API  
    - Edge Cases: SMS send failure, AI message formatting issues

  - **Sticky Note5**  
    - Role: Documentation for this block  
    - Content: "## Once the conversation is over, summarize and text lead to owner"

#### 2.4 Conversation Summarization and Lead Logging

- **Overview:**  
  After conversation completion, this block retrieves chat history from the database, summarizes the transcript and key points using AI, stores results in Google Sheets, and sends a summary SMS to the realtor.

- **Nodes Involved:**  
  - Query Supabase for conversation history (Postgres node)  
  - Combine Rows (Aggregate node)  
  - Summarize Transcript (LangChain AI Agent)  
  - Structured Output Parser1 (AI Output Parser)  
  - Store results in google (Google Sheets Append)  
  - Send Lead to owner (Twilio node)  
  - Sticky Note5 (continued documentation)  

- **Node Details:**

  - **Query Supabase for conversation history**  
    - Type: Postgres  
    - Role: Retrieves all chat messages for the current session (phone number) from Supabase database.  
    - Config: SQL query:  
      ```sql
      SELECT * FROM n8n_chat_histories WHERE session_id = '{{ $('Twilio Trigger').item.json.data.from }}';
      ```  
    - Inputs: Triggered after conversation end detection  
    - Outputs: Rows of chat history  
    - Credentials: PostgreSQL (Supabase)  
    - Edge Cases: Query failure, no history found

  - **Combine Rows**  
    - Type: Aggregate  
    - Role: Aggregates all retrieved chat history rows into a single data structure for summarization.  
    - Config: Aggregate all item data  
    - Inputs: From Postgres query  
    - Outputs: Combined conversation data  
    - Edge Cases: Empty input data

  - **Summarize Transcript**  
    - Type: LangChain AI Agent  
    - Role: Generates a clean transcript and summary of the entire conversation.  
    - Config:  
      - Input: Combined chat history data  
      - System Message: "output two fields a clean transcript of the conversation and a summary of the conversation"  
      - Output Parser enabled to enforce structured JSON output  
    - Inputs: Aggregated conversation data  
    - Outputs: JSON with `transcript` and `summary` fields  
    - Credentials: OpenAI GPT-4o-mini model via LangChain  
    - Edge Cases: AI output parsing failures, API errors

  - **Structured Output Parser1**  
    - Type: LangChain Output Parser  
    - Role: Parses AI output to structured JSON according to example schema:  
      ```json
      {
        "transcript": "transcript",
        "summary": "summary"
      }
      ```  
    - Inputs: Raw AI output from Summarize Transcript  
    - Outputs: Parsed structured data  
    - Edge Cases: Parsing failures on malformed AI output

  - **Store results in google**  
    - Type: Google Sheets Append  
    - Role: Stores the lead's phone number, conversation summary, and transcript into a Google Sheet.  
    - Config:  
      - Spreadsheet ID: `1jSlYXSBA9BJvSuMiqFUJo3omztNJtbejNYEopfbGWN8`  
      - Sheet: `gid=0` (Sheet1)  
      - Columns: Phone, Summary, Transcript  
      - Values mapped from message sender and AI output  
    - Credentials: Google Sheets OAuth2  
    - Edge Cases: API quota limits, sheet access permissions

  - **Send Lead to owner**  
    - Type: Twilio  
    - Role: Sends an SMS to the lead (or possibly the agent?) with the summarized lead info.  
    - Config:  
      - To: inbound phone number (Note: This appears to message the lead, not owner)  
      - Message: Template includes summary with "*** You just got a new lead..."  
    - Credentials: Twilio API  
    - Edge Cases: SMS failure, message clarity

  - **Sticky Note5** (also applies here)  
    - Content: "## Once the conversation is over, summarize and text lead to owner"

#### 2.5 Supportive Utility Nodes

- **Sticky Note6**  
  - Content:  
    "## AI-Powered Real Estate Lead Responder & Qualifier (n8n Workflow)\n\n** Feel free to contact me if you need help implementing (rbreen@ynteractive.com) **"  
  - Context: General workflow credit and contact

- **Sticky Note7**  
  - Content:  
    Detailed implementation instructions including links for setting up Twilio, OpenAI, Google Sheets, Supabase, and customization tips.  
  - Links:  
    - Twilio Signup: https://login.twilio.com/  
    - OpenAI API: https://platform.openai.com/  
    - Google Sheets Template: https://docs.google.com/spreadsheets/d/1jSlYXSBA9BJvSuMiqFUJo3omztNJtbejNYEopfbGWN8/edit?usp=sharing  
    - Supabase: https://supabase.com/  
  - Context: Setup guidance for users

---

### 3. Summary Table

| Node Name                        | Node Type                         | Functional Role                               | Input Node(s)                     | Output Node(s)                      | Sticky Note                                                                                                          |
|---------------------------------|----------------------------------|-----------------------------------------------|----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------------------|
| On form submission              | Form Trigger                     | Entry point; captures lead phone number       | —                                | Wait 5 Seconds                    |                                                                                                                      |
| Wait 5 Seconds                 | Wait                            | Delay before sending initial SMS               | On form submission               | Initial Text Message              |                                                                                                                      |
| Initial Text Message            | Twilio                          | Sends first SMS inviting lead to chat         | Wait 5 Seconds                  | —                                 |                                                                                                                      |
| Wait for Text Response          | Twilio Trigger                  | Listens for inbound SMS replies                 | —                                | Wait                              |                                                                                                                      |
| Wait                          | Wait                            | Delay before AI processing incoming message    | Wait for Text Response          | Real Estate Qualifier            |                                                                                                                      |
| Real Estate Qualifier           | LangChain AI Agent              | AI agent asking qualification questions        | Wait, Postgres Chat Memory      | Check if Conversation Completed  |                                                                                                                      |
| Postgres Chat Memory            | LangChain PostgreSQL Memory     | Provides chat context memory per phone number  | —                                | Real Estate Qualifier            |                                                                                                                      |
| OpenAI Chat Model2              | LangChain OpenAI Chat Model     | GPT-4o model used by AI agent                   | Real Estate Qualifier           | Real Estate Qualifier            |                                                                                                                      |
| Check if Conversation Completed | If                             | Checks if conversation ended (detects "***")  | Real Estate Qualifier           | Thank Your Text, Query Supabase / Response Text from Agent |                                                                                                                      |
| Thank Your Text                 | Twilio                          | Sends thank you SMS after conversation ends    | Check if Conversation Completed | Query Supabase for conversation history |                                                                                                                      |
| Query Supabase for conversation history | Postgres                 | Retrieves conversation history from DB          | Check if Conversation Completed | Combine Rows                    |                                                                                                                      |
| Combine Rows                   | Aggregate                       | Aggregates DB rows into single data             | Query Supabase for conversation history | Summarize Transcript          |                                                                                                                      |
| Summarize Transcript           | LangChain AI Agent              | Summarizes and transcripts conversation         | Combine Rows                   | Store results in google          |                                                                                                                      |
| Structured Output Parser1       | LangChain Output Parser         | Parses AI summary output to JSON                  | Summarize Transcript           | Store results in google          |                                                                                                                      |
| Store results in google         | Google Sheets Append            | Logs lead info into Google Sheets                 | Structured Output Parser1       | Send Lead to owner              |                                                                                                                      |
| Send Lead to owner              | Twilio                          | Sends summary SMS to lead/owner                   | Store results in google         | —                               |                                                                                                                      |
| Postgres Chat Memory           | LangChain PostgreSQL Memory     | Maintains chat memory for AI agent                | —                              | Real Estate Qualifier            |                                                                                                                      |
| OpenAI Chat Model1             | LangChain OpenAI Chat Model     | GPT-4o-mini model used for summarization          | Structured Output Parser1       | Summarize Transcript            |                                                                                                                      |
| Structured Output Parser1       | LangChain Output Parser         | Parses AI output for summary                        | Summarize Transcript            | Store results in google          |                                                                                                                      |
| Sticky Note3                   | Sticky Note                    | Documentation: Initial SMS after form submission | —                              | —                               | "## Initial Text Message after Form Submission"                                                                     |
| Sticky Note4                   | Sticky Note                    | Documentation: Wait for text reply & AI response | —                              | —                               | "## Wait for text response & Reply"                                                                                  |
| Sticky Note5                   | Sticky Note                    | Documentation: Summarize & text lead to owner     | —                              | —                               | "## Once the conversation is over, summarize and text lead to owner"                                                 |
| Sticky Note6                   | Sticky Note                    | General workflow credit and contact info          | —                              | —                               | "## AI-Powered Real Estate Lead Responder & Qualifier (n8n Workflow)\n\n** Feel free to contact me if you need help implementing (rbreen@ynteractive.com) **" |
| Sticky Note7                   | Sticky Note                    | Setup instructions with links                       | —                              | —                               | See detailed implementation steps and links (Twilio, OpenAI, Google Sheets, Supabase)                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node (Form Trigger):**  
   - Set form title: "Real Estate Inquiry"  
   - Add one form field: "Phone" (text input, required)  
   - Add form description: "Please enter your phone number with country code (example +1)"  
   - This node triggers the workflow on form submit.

2. **Add "Wait 5 Seconds" node (Wait):**  
   - Set wait time to 10 seconds.  
   - Connect output of form submission node to this wait node.

3. **Add "Initial Text Message" node (Twilio):**  
   - Configure Twilio credentials (OAuth2 or API key).  
   - Set "From" phone number to your Twilio SMS-enabled number (e.g., +17179155475).  
   - Set "To" to `{{$json.Phone}}` (phone from form submission).  
   - Message: "Thanks for submitting the form on our site realestate.com. Do you have a few minutes to answer a few questions?"  
   - Connect output of "Wait 5 Seconds" node here.

4. **Add "Wait for Text Response" node (Twilio Trigger):**  
   - Configure Twilio credentials.  
   - Listen for inbound SMS messages (`com.twilio.messaging.inbound-message.received`).  
   - This node starts the inbound SMS listening.

5. **Add "Wait" node:**  
   - Default wait (can be 0 or short delay).  
   - Connect "Wait for Text Response" output to this node.

6. **Add "Postgres Chat Memory" node (LangChain memoryPostgresChat):**  
   - Configure PostgreSQL credentials connected to your Supabase instance.  
   - Set session key to incoming phone number: `{{ $('Wait for Text Response').item.json.data.from }}` (adjust if needed).  
   - Connect output of "Wait" node to this memory node.

7. **Add "Real Estate Qualifier" node (LangChain AI Agent):**  
   - Configure OpenAI API credentials.  
   - Set model to GPT-4o.  
   - Set input text to incoming SMS body: `{{ $json.data.body }}`.  
   - Set system message to:  
     ```
     You are a friendly real estate assistant helping qualify new leads for a realtor. Ask only three brief questions, then end the conversation politely. Keep the tone warm and professional. Your goal is to gather basic information to help the realtor follow up later.

     1. Ask for their name.
     2. Ask what type of property they’re looking for (e.g., house, condo, rental).
     3. Ask what city or neighborhood they’re interested in.

     Once all three answers are collected, thank them and let them know a realtor will follow up soon. also add *** to the response
     ```
   - Connect "Wait" node output (incoming message) and "Postgres Chat Memory" to this node's inputs.

8. **Add "Check if Conversation Completed" node (If):**  
   - Condition: Check if `{{$json.output}}` contains the string `"***"`.  
   - Connect output of "Real Estate Qualifier" node to this node.

9. **Add "Thank Your Text" node (Twilio):**  
   - Configure Twilio credentials.  
   - Set "To" to incoming phone number (`{{ $('Wait for Text Response').item.json.data.from }}`).  
   - Set "From" same Twilio number.  
   - Message: "Thanks, I'll give you a call soon."  
   - Connect "true" output of If node here.

10. **Add "Query Supabase for conversation history" node (Postgres):**  
    - Configure PostgreSQL credentials.  
    - SQL query:  
      ```sql
      SELECT * FROM n8n_chat_histories WHERE session_id = '{{ $('Wait for Text Response').item.json.data.from }}';
      ```  
    - Connect "true" output of If node here (parallel branch with Thank Your Text).

11. **Add "Combine Rows" node (Aggregate):**  
    - Aggregate all items into one.  
    - Connect output of Postgres query to this node.

12. **Add "Summarize Transcript" node (LangChain AI Agent):**  
    - Configure OpenAI API credentials.  
    - Model: GPT-4o-mini.  
    - Input: combined rows data (`{{ $json.data }}`).  
    - System message: "output two fields a clean transcript of the conversation and a summary of the conversation".  
    - Enable output parser for structured JSON.  
    - Connect output of Combine Rows here.

13. **Add "Structured Output Parser1" node (LangChain Output Parser):**  
    - Define JSON schema example:  
      ```json
      {
         "transcript": "transcript",
         "summary": "summary"
      }
      ```  
    - Connect output of Summarize Transcript to this node.

14. **Add "Store results in google" node (Google Sheets):**  
    - Configure Google Sheets OAuth2 credentials.  
    - Spreadsheet ID: `1jSlYXSBA9BJvSuMiqFUJo3omztNJtbejNYEopfbGWN8`  
    - Sheet name: `gid=0` (Sheet1)  
    - Map columns:  
      - Phone: `{{ $('Wait for Text Response').item.json.data.from }}`  
      - Summary: `{{ $json.output.summary }}`  
      - Transcript: `{{ $json.output.transcript }}`  
    - Operation: Append  
    - Connect output of Structured Output Parser1 here.

15. **Add "Send Lead to owner" node (Twilio):**  
    - Configure Twilio credentials.  
    - Set "To" to lead phone number (same as Wait for Text Response).  
    - Message:  
      ```
      *** You just got a new lead. Here's a transcript of the interaction with our AI Bot. ***{{ $json.Summary }}***
      ```  
    - Connect output of Google Sheets node here.

16. **Add "Response Text from Agent" node (Twilio):**  
    - Configure Twilio credentials.  
    - Set "To" to incoming phone number (`{{ $('Wait for Text Response').item.json.data.from }}`).  
    - Message: AI output from Real Estate Qualifier node (`{{ $json.output }}`).  
    - Connect "false" output of "Check if Conversation Completed" If node here.

17. **Add supporting sticky notes for documentation:**  
    - Create sticky notes with content describing each block as per Sticky Note3, 4, 5, 6, and 7.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                              | Context or Link                                                                                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Workflow automates lead qualification by combining form submissions, SMS conversations, AI-driven questions, chat memory persistence, and lead logging via Google Sheets. Contact rbreen@ynteractive.com for implementation help.                                              | Project credit and support                                                                                                                                            |
| Setup instructions: sign up for Twilio (https://login.twilio.com/), purchase SMS-enabled number; get OpenAI API key (https://platform.openai.com/); copy Google Sheet template (https://docs.google.com/spreadsheets/d/1jSlYXSBA9BJvSuMiqFUJo3omztNJtbejNYEopfbGWN8/edit); create Supabase DB (https://supabase.com/). | Step-by-step setup guidance for all external services                                                                                                               |
| AI Agent prompt can be customized to ask different qualification questions relevant to lead funnel, e.g., loan preapproval, move-in date, etc.                                                                                                                              | Flexibility for business needs                                                                                                                                       |
| PostgreSQL database (Supabase) stores chat history for each lead keyed by phone number to maintain conversation context across messages.                                                                                                                                    | Persistence of chat memory for AI                                                                                                                                    |
| Twilio phone number must have SMS support enabled and webhook URLs properly configured in Twilio console to trigger inbound SMS events to n8n.                                                                                                                             | Twilio configuration critical for inbound message reception                                                                                                         |
| Google Sheets node appends lead summary and transcript to a shared spreadsheet for agent access.                                                                                                                                                                            | Centralized lead logging and review                                                                                                                                  |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.