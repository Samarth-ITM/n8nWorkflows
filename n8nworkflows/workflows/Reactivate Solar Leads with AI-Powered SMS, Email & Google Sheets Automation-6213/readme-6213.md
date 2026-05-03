Reactivate Solar Leads with AI-Powered SMS, Email & Google Sheets Automation

https://n8nworkflows.xyz/workflows/reactivate-solar-leads-with-ai-powered-sms--email---google-sheets-automation-6213


# Reactivate Solar Leads with AI-Powered SMS, Email & Google Sheets Automation

### 1. Workflow Overview

This workflow automates the reactivation of solar leads through an AI-enhanced multi-channel outreach sequence incorporating SMS, email, and Google Sheets data management. It targets solar sales teams or marketers aiming to re-engage inactive or unresponsive leads by systematically sending timed SMS messages and emails, while dynamically generating AI-powered SMS replies based on lead responses. The workflow includes two main logical blocks:

- **1.1 Lead Sequencing and Outreach Automation**: Periodic batch processing of leads from Google Sheets, applying filters to identify ready leads, marking campaign sequence status, and sending scheduled SMS and email messages with wait periods between each outreach step.

- **1.2 AI-Powered Incoming SMS Response Handling**: A webhook listens for inbound SMS replies, parses and identifies the lead, generates an AI-crafted response, sends it back via SMS, analyzes the intent of the response, and updates the lead status accordingly in Google Sheets.

Additional sticky notes provide setup instructions, guides, and performance tracking references.

---

### 2. Block-by-Block Analysis

#### 2.1 Lead Sequencing and Outreach Automation

- **Overview:**  
  This block periodically triggers the lead reactivation sequence by fetching lead data from Google Sheets, filtering leads ready for reactivation, and processing each lead individually through a multi-step outreach sequence involving SMS messages and emails spaced by wait nodes. It updates sequence statuses in Google Sheets to track progress.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Get row(s) in sheet (Google Sheets)  
  - Filter Ready Leads (Code)  
  - Process One Lead at a Time (SplitInBatches)  
  - Mark Sequence Active (Google Sheets)  
  - Week 1 SMS (Twilio)  
  - Wait 1 Week (Wait)  
  - Week 2 SMS (Twilio)  
  - Wait 2 Weeks (Wait)  
  - Week 3 Email (Email Send)  
  - Wait 3 Weeks (Wait)  
  - Week 4 SMS (Twilio)  
  - Wait 4 Weeks (Wait)  
  - Final SMS (Twilio)  
  - Mark Sequence Complete (Google Sheets)

- **Node Details:**

  1. **Schedule Trigger**  
     - *Type:* Schedule Trigger  
     - *Role:* Initiates the workflow on a defined schedule (e.g., daily or weekly)  
     - *Key Config:* Cron or interval-based trigger (exact schedule not specified in JSON)  
     - *Inputs:* None  
     - *Outputs:* Leads to Google Sheets data retrieval  
     - *Potential Failures:* Missed schedules, time zone issues

  2. **Get row(s) in sheet**  
     - *Type:* Google Sheets (Read)  
     - *Role:* Retrieves lead records from a specific sheet  
     - *Config:* Reads all or filtered rows as input data  
     - *Inputs:* Trigger from Schedule Trigger  
     - *Outputs:* Array of lead data rows  
     - *Failures:* Authentication errors, API rate limits, empty data sets

  3. **Filter Ready Leads**  
     - *Type:* Code (JavaScript)  
     - *Role:* Filters lead rows to find those eligible for reactivation based on criteria (e.g., lead status, last contact date)  
     - *Config:* Custom script that processes input data and outputs filtered leads  
     - *Inputs:* Lead rows from Google Sheets  
     - *Outputs:* Filtered lead subset for processing  
     - *Failures:* Script errors, unexpected data formats

  4. **Process One Lead at a Time**  
     - *Type:* SplitInBatches  
     - *Role:* Processes leads sequentially to prevent API overload and maintain state integrity  
     - *Config:* Batch size of 1 (one lead per execution)  
     - *Inputs:* Filtered leads  
     - *Outputs:* Individual lead data for next steps  
     - *Failures:* Batch processing errors, workflow delays

  5. **Mark Sequence Active**  
     - *Type:* Google Sheets (Update)  
     - *Role:* Marks the lead's sequence status as active, indicating outreach has started  
     - *Config:* Updates designated cell/column in Google Sheet  
     - *Inputs:* Single lead from batch  
     - *Outputs:* Confirmation of update, triggers Week 1 SMS  
     - *Failures:* API errors, incorrect row identification

  6. **Week 1 SMS**  
     - *Type:* Twilio (Send SMS)  
     - *Role:* Sends first SMS message to lead  
     - *Config:* Uses Twilio credentials, message template possibly including lead personalization  
     - *Inputs:* Lead phone number, message content  
     - *Outputs:* Triggers Wait 1 Week node  
     - *Failures:* Twilio auth issues, invalid phone number, SMS send failures

  7. **Wait 1 Week**  
     - *Type:* Wait  
     - *Role:* Pauses workflow for 1 week before next outreach  
     - *Config:* Fixed 7-day wait  
     - *Inputs:* After Week 1 SMS  
     - *Outputs:* Triggers Week 2 SMS  
     - *Failures:* Workflow timeouts, delays on server

  8. **Week 2 SMS**  
     - *Type:* Twilio (Send SMS)  
     - *Role:* Sends second SMS message  
     - *Config:* Similar to Week 1 SMS with different content  
     - *Inputs:* Lead data  
     - *Outputs:* Triggers Wait 2 Weeks  
     - *Failures:* As Week 1 SMS

  9. **Wait 2 Weeks**  
     - *Type:* Wait  
     - *Role:* Pauses for 2 weeks before next outreach  
     - *Config:* 14-day wait  
     - *Inputs:* After Week 2 SMS  
     - *Outputs:* Triggers Week 3 Email  
     - *Failures:* Same as previous waits

  10. **Week 3 Email**  
      - *Type:* Email Send  
      - *Role:* Sends an email outreach to the lead  
      - *Config:* Uses configured SMTP or email credentials, personalized content  
      - *Inputs:* Lead email address, email body  
      - *Outputs:* Triggers Wait 3 Weeks  
      - *Failures:* Email auth, invalid addresses, SMTP issues

  11. **Wait 3 Weeks**  
      - *Type:* Wait  
      - *Role:* Pauses for 3 weeks before next SMS  
      - *Config:* 21-day wait  
      - *Inputs:* After Week 3 Email  
      - *Outputs:* Triggers Week 4 SMS  
      - *Failures:* Same as above waits

  12. **Week 4 SMS**  
      - *Type:* Twilio (Send SMS)  
      - *Role:* Sends fourth SMS message  
      - *Config:* Twilio with updated message content  
      - *Inputs:* Lead phone number  
      - *Outputs:* Triggers Wait 4 Weeks  
      - *Failures:* Twilio or phone issues

  13. **Wait 4 Weeks**  
      - *Type:* Wait  
      - *Role:* Pauses for 4 weeks before final SMS outreach  
      - *Config:* 28-day wait  
      - *Inputs:* After Week 4 SMS  
      - *Outputs:* Triggers Final SMS  
      - *Failures:* Workflow timing delays

  14. **Final SMS**  
      - *Type:* Twilio (Send SMS)  
      - *Role:* Sends a last SMS message as final outreach  
      - *Config:* Twilio SMS, possibly with urgency or closing message  
      - *Inputs:* Lead phone number  
      - *Outputs:* Triggers Mark Sequence Complete  
      - *Failures:* Twilio errors

  15. **Mark Sequence Complete**  
      - *Type:* Google Sheets (Update)  
      - *Role:* Updates lead status to indicate sequence completion  
      - *Config:* Updates relevant row and column in Google Sheets  
      - *Inputs:* Lead data after final SMS  
      - *Outputs:* End of sequence processing for the lead  
      - *Failures:* API write errors

---

#### 2.2 AI-Powered Incoming SMS Response Handling

- **Overview:**  
  This block listens for inbound SMS replies via a webhook, parses the message and sender information, finds the corresponding lead data, generates an AI-based reply using OpenAI, sends this AI reply back via Twilio SMS, analyzes the intent of the response, and updates Google Sheets with the new response status.

- **Nodes Involved:**  
  - SMS Reply Webhook  
  - Parse Reply (Code)  
  - Find Lead Data (Google Sheets)  
  - AI Generate Response (OpenAI)  
  - Send AI Reply (Twilio)  
  - Analyze Response Intent (Code)  
  - Update Response Status (Google Sheets)

- **Node Details:**

  1. **SMS Reply Webhook**  
     - *Type:* Webhook  
     - *Role:* Receives inbound SMS messages from Twilio or similar SMS gateway  
     - *Config:* HTTP POST endpoint configured on Twilio for inbound SMS  
     - *Inputs:* Incoming SMS payload (phone number, message body, metadata)  
     - *Outputs:* Raw inbound SMS data to Parse Reply node  
     - *Failures:* Webhook unreachable, payload format changes

  2. **Parse Reply**  
     - *Type:* Code (JavaScript)  
     - *Role:* Extracts relevant fields (e.g., sender phone, message text) and normalizes data for downstream processing  
     - *Config:* Custom script to parse webhook payload  
     - *Inputs:* Raw webhook data  
     - *Outputs:* Cleaned data for lead lookup  
     - *Failures:* Parsing errors, unexpected payload structure

  3. **Find Lead Data**  
     - *Type:* Google Sheets (Read)  
     - *Role:* Searches Google Sheets for lead matching inbound phone number  
     - *Config:* Query or filter based on parsed phone number  
     - *Inputs:* Parsed phone from SMS  
     - *Outputs:* Lead record data for AI response generation  
     - *Failures:* No matching lead found, API errors

  4. **AI Generate Response**  
     - *Type:* OpenAI  
     - *Role:* Generates a tailored SMS reply based on inbound message and lead context  
     - *Config:* Uses OpenAI credentials, prompt engineering to elicit appropriate responses  
     - *Inputs:* Lead data and inbound message content  
     - *Outputs:* AI-generated SMS reply text  
     - *Failures:* API rate limits, authentication, prompt errors

  5. **Send AI Reply**  
     - *Type:* Twilio (Send SMS)  
     - *Role:* Sends the AI-generated reply back to the lead’s phone number  
     - *Config:* Uses Twilio credentials and AI response content  
     - *Inputs:* Phone number, AI reply message  
     - *Outputs:* Triggers intent analysis  
     - *Failures:* Twilio errors, invalid phone

  6. **Analyze Response Intent**  
     - *Type:* Code (JavaScript)  
     - *Role:* Processes AI reply and/or inbound message to determine lead intent (e.g., interested, stop, no response)  
     - *Config:* Custom logic or NLP expressions  
     - *Inputs:* AI reply and inbound message data  
     - *Outputs:* Intent classification for updating lead status  
     - *Failures:* Logic errors, misclassification

  7. **Update Response Status**  
     - *Type:* Google Sheets (Update)  
     - *Role:* Updates the lead record with the latest response status and intent  
     - *Config:* Writes to specific columns in the Google Sheet  
     - *Inputs:* Lead ID and intent data  
     - *Outputs:* Confirmation of update; ends AI response cycle  
     - *Failures:* API write errors

---

### 3. Summary Table

| Node Name             | Node Type           | Functional Role                      | Input Node(s)                | Output Node(s)                | Sticky Note                                                  |
|-----------------------|---------------------|-----------------------------------|-----------------------------|------------------------------|--------------------------------------------------------------|
| 📋 Workflow Overview  | Sticky Note         | Documentation                     | —                           | —                            |                                                              |
| 📊 Google Sheets Guide | Sticky Note         | Documentation                     | —                           | —                            |                                                              |
| 📅 Sequence Strategy   | Sticky Note         | Documentation                     | —                           | —                            |                                                              |
| 🤖 AI Response Guide   | Sticky Note         | Documentation                     | —                           | —                            |                                                              |
| ⚙️ Setup Instructions   | Sticky Note         | Documentation                     | —                           | —                            |                                                              |
| 📈 Performance Guide   | Sticky Note         | Documentation                     | —                           | —                            |                                                              |
| Schedule Trigger      | Schedule Trigger     | Initiates scheduled execution      | —                           | Get row(s) in sheet          |                                                              |
| Get row(s) in sheet   | Google Sheets       | Reads lead data                   | Schedule Trigger             | Filter Ready Leads           |                                                              |
| Filter Ready Leads    | Code                | Filters leads ready for outreach  | Get row(s) in sheet          | Process One Lead at a Time   |                                                              |
| Process One Lead at a Time | SplitInBatches  | Processes leads sequentially      | Filter Ready Leads           | (empty branch), Mark Sequence Active |                                                              |
| Mark Sequence Active  | Google Sheets       | Marks lead sequence as active     | Process One Lead at a Time   | Week 1 SMS                  |                                                              |
| Week 1 SMS            | Twilio              | Sends first SMS outreach          | Mark Sequence Active         | Wait 1 Week                 |                                                              |
| Wait 1 Week           | Wait                | Waits 1 week before next step     | Week 1 SMS                  | Week 2 SMS                  |                                                              |
| Week 2 SMS            | Twilio              | Sends second SMS outreach         | Wait 1 Week                 | Wait 2 Weeks                |                                                              |
| Wait 2 Weeks          | Wait                | Waits 2 weeks before next step    | Week 2 SMS                  | Week 3 Email                |                                                              |
| Week 3 Email          | Email Send          | Sends email outreach              | Wait 2 Weeks                | Wait 3 Weeks                |                                                              |
| Wait 3 Weeks          | Wait                | Waits 3 weeks before next step    | Week 3 Email                | Week 4 SMS                  |                                                              |
| Week 4 SMS            | Twilio              | Sends fourth SMS outreach         | Wait 3 Weeks                | Wait 4 Weeks                |                                                              |
| Wait 4 Weeks          | Wait                | Waits 4 weeks before final SMS    | Week 4 SMS                  | Final SMS                   |                                                              |
| Final SMS             | Twilio              | Sends final SMS outreach          | Wait 4 Weeks                | Mark Sequence Complete      |                                                              |
| Mark Sequence Complete | Google Sheets       | Marks sequence as complete        | Final SMS                   | —                          |                                                              |
| SMS Reply Webhook     | Webhook             | Receives inbound SMS replies      | —                           | Parse Reply                 |                                                              |
| Parse Reply           | Code                | Parses inbound SMS data           | SMS Reply Webhook            | Find Lead Data              |                                                              |
| Find Lead Data        | Google Sheets       | Finds lead matching inbound SMS   | Parse Reply                 | AI Generate Response        |                                                              |
| AI Generate Response  | OpenAI              | Creates AI SMS reply              | Find Lead Data              | Send AI Reply               |                                                              |
| Send AI Reply         | Twilio              | Sends AI-generated SMS            | AI Generate Response        | Analyze Response Intent     |                                                              |
| Analyze Response Intent | Code              | Determines intent from replies    | Send AI Reply               | Update Response Status      |                                                              |
| Update Response Status | Google Sheets       | Updates lead's response status    | Analyze Response Intent     | —                          |                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Schedule Trigger node:**
   - Set to run on desired frequency (e.g., daily at a specific time).
   - This node triggers the lead reactivation sequence.

3. **Add a Google Sheets node ("Get row(s) in sheet"):**
   - Configure credentials for Google Sheets API.
   - Set to read the entire leads sheet holding all solar leads.
   - Connect Schedule Trigger output to this node input.

4. **Add a Code node ("Filter Ready Leads"):**
   - Use JavaScript to filter leads based on criteria such as sequence status = "ready" or last contact date older than threshold.
   - Input: data from Google Sheets node.
   - Output: filtered leads array.

5. **Add a SplitInBatches node ("Process One Lead at a Time"):**
   - Batch size: 1.
   - Input: filtered leads from Code node.
   - This will process each lead independently.

6. **Add a Google Sheets node ("Mark Sequence Active"):**
   - Configure to update the lead’s row in Google Sheets, setting sequence status to "active".
   - Input: from SplitInBatches node (successful batch).
   - Output: triggers Week 1 SMS node.

7. **Add a Twilio node ("Week 1 SMS"):**
   - Configure Twilio credentials.
   - Set message body for first outreach SMS.
   - Phone number dynamically set from lead data.
   - Connect Mark Sequence Active node output to this node.

8. **Add a Wait node ("Wait 1 Week"):**
   - Duration: 1 week (7 days).
   - Connect Week 1 SMS output here.

9. **Add a Twilio node ("Week 2 SMS"):**
   - Second outreach SMS content.
   - Connect Wait 1 Week output here.

10. **Add a Wait node ("Wait 2 Weeks"):**
    - Duration: 14 days.
    - Connect Week 2 SMS output here.

11. **Add an Email Send node ("Week 3 Email"):**
    - Configure SMTP/email credentials.
    - Compose email message for third outreach.
    - Use lead’s email from data.
    - Connect Wait 2 Weeks output here.

12. **Add a Wait node ("Wait 3 Weeks"):**
    - Duration: 21 days.
    - Connect Week 3 Email output here.

13. **Add a Twilio node ("Week 4 SMS"):**
    - Fourth outreach SMS content.
    - Connect Wait 3 Weeks output here.

14. **Add a Wait node ("Wait 4 Weeks"):**
    - Duration: 28 days.
    - Connect Week 4 SMS output here.

15. **Add a Twilio node ("Final SMS"):**
    - Final outreach SMS message.
    - Connect Wait 4 Weeks output here.

16. **Add a Google Sheets node ("Mark Sequence Complete"):**
    - Updates lead sequence status to "complete".
    - Connect Final SMS output here.

17. **Create a Webhook node ("SMS Reply Webhook"):**
    - Configure HTTP POST endpoint URL.
    - This endpoint will receive inbound SMS from Twilio.

18. **Add a Code node ("Parse Reply"):**
    - Parse incoming webhook payload extracting phone number and message text.
    - Connect Webhook output to this node.

19. **Add a Google Sheets node ("Find Lead Data"):**
    - Configure to lookup lead by phone number from parsed reply.
    - Connect Parse Reply output here.

20. **Add an OpenAI node ("AI Generate Response"):**
    - Configure OpenAI credentials.
    - Use prompt template to generate contextual SMS reply based on inbound message and lead data.
    - Input from Find Lead Data.
    - Output to next node.

21. **Add a Twilio node ("Send AI Reply"):**
    - Sends AI-generated SMS reply to lead's number.
    - Connect OpenAI output here.

22. **Add a Code node ("Analyze Response Intent"):**
    - Analyze reply or inbound message to classify lead intent (e.g., interested, stop, no response).
    - Connect Send AI Reply output here.

23. **Add a Google Sheets node ("Update Response Status"):**
    - Update lead record with response status and intent classification.
    - Connect Analyze Response Intent output here.

24. **Connect all nodes as specified to maintain the logical flow.**

25. **Configure credentials securely for Google Sheets, Twilio, Email, and OpenAI.**

26. **Test each step with sample data to validate correct operation and error handling.**

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                              |
|-----------------------------------------------------------------------------------------------|----------------------------------------------|
| Setup instructions, Google Sheets guide, AI response guide, sequence strategy, and performance guide are embedded as sticky notes within the workflow for easy reference during use and modification. | Internal workflow documentation              |
| To enable inbound SMS handling, configure Twilio webhook URL to point to the "SMS Reply Webhook" node’s URL in n8n. | Twilio webhook setup                         |
| Ensure OpenAI API key has sufficient quota and permissions for conversational AI text generation. | OpenAI account management                    |
| Recommended to monitor API rate limits on Google Sheets, Twilio, and OpenAI to avoid workflow failures. | API usage best practices                      |
| The workflow uses staged wait periods to space out outreach, reducing contact fatigue and improving engagement chances. | Outreach strategy best practice               |

---

**Disclaimer:** The text provided is derived exclusively from an n8n automated workflow. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.