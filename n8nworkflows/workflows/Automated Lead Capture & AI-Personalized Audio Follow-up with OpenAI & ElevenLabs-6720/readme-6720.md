Automated Lead Capture & AI-Personalized Audio Follow-up with OpenAI & ElevenLabs

https://n8nworkflows.xyz/workflows/automated-lead-capture---ai-personalized-audio-follow-up-with-openai---elevenlabs-6720


# Automated Lead Capture & AI-Personalized Audio Follow-up with OpenAI & ElevenLabs

### 1. Workflow Overview

This workflow automates the capture, AI-driven qualification, and personalized audio follow-up of new leads, integrating OpenAI for lead assessment and ElevenLabs for audio generation. It targets sales and marketing teams who want to automate lead qualification and enhance engagement by sending personalized audio messages. The workflow logically divides into these blocks:

- **1.1 Input Reception:** Capture new lead data via webhook and clean it.
- **1.2 AI Qualification & Summary:** Use OpenAI to analyze lead quality and generate a summary.
- **1.3 Qualification Decision:** Determine if the lead is qualified based on AI output.
- **1.4 Personalized Audio Creation:** For qualified leads, generate and upload personalized audio using ElevenLabs and Google Drive.
- **1.5 CRM Integration:** Check for lead duplicates in CRM and create or update lead entries accordingly.
- **1.6 Follow-up & Notifications:** Send personalized follow-up emails, notify the sales team via Slack, and log lead details in Google Sheets.
- **1.7 Handling Unqualified Leads:** Mark and log unqualified leads appropriately.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Captures incoming lead data from an external source via webhook and prepares it for processing.
- **Nodes Involved:** 
  - New Lead Captured
  - Extract & Clean Lead Data

- **Node Details:**

  - **New Lead Captured**
    - Type: Webhook
    - Role: Entry point capturing lead submission data in real-time.
    - Configuration: Uses a unique webhook URL, no additional parameters.
    - Inputs: External HTTP request with lead data.
    - Outputs: Passes raw lead data to next node.
    - Edge Cases: Missing or malformed webhook requests; network timeouts.
  
  - **Extract & Clean Lead Data**
    - Type: Set
    - Role: Extracts and sanitizes key lead data fields (e.g., name, email, phone, message).
    - Configuration: Maps raw input fields to clean variables for downstream use.
    - Inputs: From webhook node.
    - Outputs: Cleaned lead data forwarded to AI processing.
    - Edge Cases: Missing expected fields; incorrect data formats.

#### 2.2 AI Qualification & Summary

- **Overview:** Uses OpenAI’s language model to analyze the lead’s data, qualify the lead, and generate a summary.
- **Nodes Involved:** 
  - AI Qualification & Summary
  - Parse AI Qualification

- **Node Details:**

  - **AI Qualification & Summary**
    - Type: OpenAI (via LangChain node)
    - Role: Sends lead data prompt to OpenAI for analysis.
    - Configuration: Uses OpenAI API with a prompt designed to qualify leads and produce a lead summary.
    - Inputs: Cleaned lead data.
    - Outputs: AI-generated JSON with qualification and summary details.
    - Edge Cases: API rate limits; invalid API keys; malformed responses.
  
  - **Parse AI Qualification**
    - Type: Set
    - Role: Parses OpenAI output to extract qualification flags and summary text into usable variables.
    - Configuration: Maps AI output fields like `aiIsQualified`, `aiSummary`.
    - Inputs: AI Qualification node output.
    - Outputs: Data for decision-making node.
    - Edge Cases: Unexpected AI response formats; missing keys.

#### 2.3 Qualification Decision

- **Overview:** Determines if the lead qualifies for personalized follow-up based on AI output, branching workflow accordingly.
- **Nodes Involved:** 
  - Is Lead Qualified?
  - Mark Lead as Unqualified

- **Node Details:**

  - **Is Lead Qualified?**
    - Type: If
    - Role: Conditional branching based on `aiIsQualified` value.
    - Configuration: Checks if AI qualification flag is true.
    - Inputs: Parsed AI qualification data.
    - Outputs: 
      - True: Proceed to personalized audio creation.
      - False: Mark lead as unqualified.
    - Edge Cases: Missing or invalid qualification flags.
  
  - **Mark Lead as Unqualified**
    - Type: Set
    - Role: Sets lead status to unqualified for logging and stops personalized follow-up.
    - Configuration: Adds fields such as `aiIsQualified = false`.
    - Inputs: False branch from qualification check.
    - Outputs: Leads to logging node.
    - Edge Cases: None significant.

#### 2.4 Personalized Audio Creation

- **Overview:** For qualified leads, crafts personalized audio text, generates audio via ElevenLabs, uploads audio to Google Drive, and stores the file link.
- **Nodes Involved:** 
  - Craft Personalized Audio Text
  - Generate Personalized Audio
  - Upload Personalized Audio
  - Store Personalized Audio Link

- **Node Details:**

  - **Craft Personalized Audio Text**
    - Type: Code
    - Role: Generates a personalized audio script using lead data and AI summary.
    - Configuration: JavaScript code concatenates and formats text for audio generation.
    - Inputs: Qualified lead data and AI summary.
    - Outputs: Text string for audio synthesis.
    - Edge Cases: Missing variables causing script failure.
  
  - **Generate Personalized Audio**
    - Type: ElevenLabs
    - Role: Converts text to speech using ElevenLabs API.
    - Configuration: Uses ElevenLabs credentials; voice and audio format parameters set.
    - Inputs: Personalized audio text.
    - Outputs: Audio file data.
    - Edge Cases: API failures, invalid API keys, network errors.
  
  - **Upload Personalized Audio**
    - Type: Google Drive
    - Role: Uploads generated audio file to Google Drive for storage.
    - Configuration: Uses Google Drive OAuth2 credentials; target folder specified.
    - Inputs: Audio file data.
    - Outputs: Google Drive file metadata including public URL.
    - Edge Cases: Permission errors; upload failures.
  
  - **Store Personalized Audio Link**
    - Type: Set
    - Role: Saves the Google Drive audio file URL in workflow data for CRM and follow-up.
    - Configuration: Maps Drive URL to `personalizedAudioUrl` variable.
    - Inputs: Upload node output.
    - Outputs: Leads to CRM duplicate check.
    - Edge Cases: Missing URL or failed uploads.

#### 2.5 CRM Integration

- **Overview:** Checks if lead already exists in CRM, then updates or creates a lead record accordingly.
- **Nodes Involved:** 
  - Check CRM for Duplicate Lead
  - Is Lead a Duplicate?
  - Update Lead in CRM
  - Create Lead in CRM
  - CRM Result

- **Node Details:**

  - **Check CRM for Duplicate Lead**
    - Type: HTTP Request
    - Role: Queries CRM system API to find if lead email or phone already exists.
    - Configuration: HTTP GET or POST with authentication headers; query parameters based on lead data.
    - Inputs: Lead data including email or phone.
    - Outputs: CRM API response with lead existence status.
    - Edge Cases: API downtime, authentication errors, data format issues.
  
  - **Is Lead a Duplicate?**
    - Type: If
    - Role: Branches workflow based on CRM query result.
    - Configuration: Evaluates if CRM response indicates duplicate.
    - Inputs: CRM check output.
    - Outputs:
      - True: Update existing lead.
      - False: Create new lead.
    - Edge Cases: Ambiguous CRM responses.
  
  - **Update Lead in CRM**
    - Type: HTTP Request
    - Role: Sends update request to CRM with new lead data and personalized audio URL.
    - Configuration: HTTP PUT or PATCH with necessary headers and payload.
    - Inputs: Duplicate lead branch.
    - Outputs: Passes result to merge node.
    - Edge Cases: API failures, partial updates.
  
  - **Create Lead in CRM**
    - Type: HTTP Request
    - Role: Sends new lead creation request to CRM.
    - Configuration: HTTP POST with lead data and audio URL.
    - Inputs: Non-duplicate branch.
    - Outputs: Passes result to merge node.
    - Edge Cases: API failures, duplicate creation race conditions.
  
  - **CRM Result**
    - Type: Merge
    - Role: Combines update or creation results for unified downstream processing.
    - Configuration: Merges multiple input streams.
    - Inputs: From Update and Create nodes.
    - Outputs: Leads to email sending node.
    - Edge Cases: Merge conflicts or empty inputs.

#### 2.6 Follow-up & Notifications

- **Overview:** Sends a personalized email follow-up with audio attachment, notifies sales team via Slack, and logs all lead data including AI and CRM statuses.
- **Nodes Involved:** 
  - Send Personalized Follow-up
  - Notify Sales Team
  - Log Lead Data

- **Node Details:**

  - **Send Personalized Follow-up**
    - Type: Gmail
    - Role: Sends an email to the lead with personalized audio and AI summary.
    - Configuration: Uses Gmail OAuth2 credentials; email template includes audio link.
    - Inputs: Merged CRM result and personalized audio data.
    - Outputs: Leads to sales notification.
    - Edge Cases: Email delivery failures, authentication errors.
  
  - **Notify Sales Team**
    - Type: Slack
    - Role: Posts a notification message in Slack channel alerting sales about the new qualified lead and follow-up.
    - Configuration: Slack webhook URL or OAuth2; message includes lead summary and status.
    - Inputs: Email sent confirmation.
    - Outputs: Leads to logging node.
    - Edge Cases: Slack API issues, invalid webhook.
  
  - **Log Lead Data**
    - Type: Google Sheets
    - Role: Logs all lead details, AI qualification, CRM status, audio URL, and follow-up information into a spreadsheet for record-keeping.
    - Configuration: Google Sheets OAuth2 credentials; uses expressions to map variables to columns.
    - Inputs: Notification completion.
    - Outputs: End of workflow.
    - Edge Cases: Permission issues, quota limits.

#### 2.7 Handling Unqualified Leads

- **Overview:** For leads not qualified by AI, marks status accordingly and logs the data without proceeding to audio or CRM steps.
- **Nodes Involved:** 
  - Mark Lead as Unqualified
  - Log Lead Data

- **Node Details:**  
  (Covered above in qualification decision and logging.)

---

### 3. Summary Table

| Node Name                   | Node Type                   | Functional Role                         | Input Node(s)                    | Output Node(s)                   | Sticky Note                                                  |
|-----------------------------|-----------------------------|---------------------------------------|---------------------------------|---------------------------------|--------------------------------------------------------------|
| New Lead Captured            | Webhook                     | Entry point for lead data              | —                               | Extract & Clean Lead Data        |                                                              |
| Extract & Clean Lead Data    | Set                         | Clean and prepare lead data            | New Lead Captured               | AI Qualification & Summary       |                                                              |
| AI Qualification & Summary  | OpenAI (LangChain)          | Analyze and summarize lead             | Extract & Clean Lead Data       | Parse AI Qualification           |                                                              |
| Parse AI Qualification       | Set                         | Parse AI output                        | AI Qualification & Summary      | Is Lead Qualified?               |                                                              |
| Is Lead Qualified?           | If                          | Branch based on lead qualification    | Parse AI Qualification          | Craft Personalized Audio Text / Mark Lead as Unqualified |                                                              |
| Craft Personalized Audio Text| Code                        | Generate personalized audio script    | Is Lead Qualified? (true)       | Generate Personalized Audio      |                                                              |
| Generate Personalized Audio  | ElevenLabs                  | Convert text to speech                 | Craft Personalized Audio Text   | Upload Personalized Audio        |                                                              |
| Upload Personalized Audio    | Google Drive                | Store audio file                       | Generate Personalized Audio     | Store Personalized Audio Link    |                                                              |
| Store Personalized Audio Link| Set                         | Save audio URL for CRM & follow-up    | Upload Personalized Audio       | Check CRM for Duplicate Lead     |                                                              |
| Check CRM for Duplicate Lead | HTTP Request                | Query CRM for existing lead            | Store Personalized Audio Link   | Is Lead a Duplicate?             |                                                              |
| Is Lead a Duplicate?         | If                          | Branch: update or create lead in CRM  | Check CRM for Duplicate Lead    | Update Lead in CRM / Create Lead in CRM |                                                              |
| Update Lead in CRM           | HTTP Request                | Update existing lead record            | Is Lead a Duplicate? (true)     | CRM Result                      |                                                              |
| Create Lead in CRM           | HTTP Request                | Create new lead record                 | Is Lead a Duplicate? (false)    | CRM Result                      |                                                              |
| CRM Result                  | Merge                       | Combine CRM operation results          | Update Lead in CRM, Create Lead in CRM | Send Personalized Follow-up |                                                              |
| Send Personalized Follow-up  | Gmail                       | Email follow-up with audio             | CRM Result                     | Notify Sales Team                |                                                              |
| Notify Sales Team            | Slack                       | Notify sales team of lead and follow-up| Send Personalized Follow-up    | Log Lead Data                   |                                                              |
| Log Lead Data                | Google Sheets               | Log all lead details                   | Notify Sales Team / Mark Lead as Unqualified | —                           | Values: Timestamp, Lead Info, AI Qualification, Audio URL, CRM status, Follow-up flag, Status (Success/Error) |
| Mark Lead as Unqualified     | Set                         | Mark leads unqualified                  | Is Lead Qualified? (false)      | Log Lead Data                   |                                                              |
| Sticky Note                 | Sticky Note                 | —                                     | —                               | —                               | Covers empty content, no content present                     |
| Sticky Note1                | Sticky Note                 | —                                     | —                               | —                               | Covers empty content, no content present                     |
| Sticky Note2                | Sticky Note                 | —                                     | —                               | —                               | Covers empty content, no content present                     |
| Sticky Note3                | Sticky Note                 | —                                     | —                               | —                               | Covers empty content, no content present                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node:**
   - Type: Webhook
   - Name: "New Lead Captured"
   - Set unique webhook URL for incoming lead data.
   - No additional parameters.

2. **Add Set Node to Clean Lead Data:**
   - Name: "Extract & Clean Lead Data"
   - Map incoming webhook data fields (e.g., `leadName`, `leadEmail`, `leadPhone`, `leadMessage`) into clean variables.
   - Connect output of Webhook node to this node.

3. **Add OpenAI Node for Qualification:**
   - Name: "AI Qualification & Summary"
   - Type: OpenAI via LangChain
   - Configure OpenAI credentials (API key).
   - Set prompt template to analyze lead data and return JSON with `aiIsQualified` (boolean) and `aiSummary` (string).
   - Connect from "Extract & Clean Lead Data".

4. **Add Set Node to Parse AI Output:**
   - Name: "Parse AI Qualification"
   - Map OpenAI JSON fields (`aiIsQualified`, `aiSummary`) into workflow variables.
   - Connect from "AI Qualification & Summary".

5. **Add If Node to Check Qualification:**
   - Name: "Is Lead Qualified?"
   - Condition: Check if `aiIsQualified` == true.
   - Connect from "Parse AI Qualification".
   - True branch: proceed to personalized audio creation.
   - False branch: proceed to mark as unqualified.

6. **Add Set Node for Unqualified Leads:**
   - Name: "Mark Lead as Unqualified"
   - Set `aiIsQualified = false` and any relevant flags.
   - Connect from False branch of "Is Lead Qualified?".
   - Connect output to "Log Lead Data" node (to be created).

7. **Create Code Node for Audio Text:**
   - Name: "Craft Personalized Audio Text"
   - Script: Compose personalized message using lead info and AI summary.
   - Connect from True branch of "Is Lead Qualified?".

8. **Add ElevenLabs Text-to-Speech Node:**
   - Name: "Generate Personalized Audio"
   - Configure ElevenLabs API credentials.
   - Set parameters for voice style and output format.
   - Input text from "Craft Personalized Audio Text".
   - Connect from previous node.

9. **Add Google Drive Upload Node:**
   - Name: "Upload Personalized Audio"
   - Configure Google Drive OAuth2 credentials.
   - Target folder for audio files set.
   - Upload audio file from ElevenLabs node.
   - Connect from "Generate Personalized Audio".

10. **Add Set Node to Store Audio URL:**
    - Name: "Store Personalized Audio Link"
    - Map Google Drive file URL to variable `personalizedAudioUrl`.
    - Connect from "Upload Personalized Audio".

11. **Add HTTP Request Node to Check CRM for Duplicate Lead:**
    - Name: "Check CRM for Duplicate Lead"
    - Configure HTTP request with CRM API endpoint to query lead by email or phone.
    - Set authentication headers as needed.
    - Connect from "Store Personalized Audio Link".

12. **Add If Node to Branch by Duplicate Status:**
    - Name: "Is Lead a Duplicate?"
    - Condition: Determine duplicate by examining CRM response.
    - Connect from "Check CRM for Duplicate Lead".
    - True branch: Update existing lead.
    - False branch: Create new lead.

13. **Add HTTP Request Node to Update Lead in CRM:**
    - Name: "Update Lead in CRM"
    - Configure HTTP PUT/PATCH request to CRM API with updated lead data and audio URL.
    - Set authentication.
    - Connect from True branch of "Is Lead a Duplicate?".

14. **Add HTTP Request Node to Create Lead in CRM:**
    - Name: "Create Lead in CRM"
    - Configure HTTP POST request to CRM API with lead data and audio URL.
    - Set authentication.
    - Connect from False branch of "Is Lead a Duplicate?".

15. **Add Merge Node for CRM Result:**
    - Name: "CRM Result"
    - Merge inputs from "Update Lead in CRM" and "Create Lead in CRM".
    - Connect outputs of both nodes to this node.

16. **Add Gmail Node to Send Follow-up:**
    - Name: "Send Personalized Follow-up"
    - Configure Gmail OAuth2 credentials.
    - Compose email template including personalized audio URL and AI summary.
    - Connect from "CRM Result".

17. **Add Slack Node to Notify Sales Team:**
    - Name: "Notify Sales Team"
    - Configure Slack webhook or OAuth2.
    - Message contains lead info, qualification, and follow-up status.
    - Connect from "Send Personalized Follow-up".

18. **Add Google Sheets Node to Log Lead Data:**
    - Name: "Log Lead Data"
    - Configure Google Sheets OAuth2.
    - Map all relevant fields:
      - Timestamp: `={{ new Date().toISOString() }}`
      - Lead Name, Email, Phone, Message
      - AI Qualification and Summary
      - Qualification status flag
      - Personalized Audio URL (or "N/A" if unqualified)
      - CRM Status (default "Not Processed")
      - Follow-up Sent flag ("Yes"/"No")
      - Overall Status (Success/Error)
    - Connect from "Notify Sales Team" and from "Mark Lead as Unqualified".

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                               |
|-----------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Logging includes detailed fields for traceability: Timestamp, AI results, CRM status, etc.    | Facilitates auditing and troubleshooting of lead processing. |
| ElevenLabs provides high-quality voice synthesis for personalized outreach.                   | https://elevenlabs.io                                          |
| OpenAI integration relies on LangChain node for flexible prompt engineering and parsing.      | https://openai.com                                            |
| Google Sheets logging supports monitoring lead flow and success metrics.                      |                                                               |
| Slack notifications ensure real-time alerts for sales teams improving responsiveness.        |                                                               |
| Gmail node requires OAuth2 credentials with "send email" permission.                          |                                                               |

---

This comprehensive documentation provides a full understanding of the workflow’s structure, logic, and integration points, enabling reproduction, modification, and error anticipation.