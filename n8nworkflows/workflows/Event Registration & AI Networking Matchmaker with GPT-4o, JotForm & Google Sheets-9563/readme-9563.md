Event Registration & AI Networking Matchmaker with GPT-4o, JotForm & Google Sheets

https://n8nworkflows.xyz/workflows/event-registration---ai-networking-matchmaker-with-gpt-4o--jotform---google-sheets-9563


# Event Registration & AI Networking Matchmaker with GPT-4o, JotForm & Google Sheets

### 1. Workflow Overview

This workflow automates event registration processing and attendee networking facilitation using AI profiling. It targets event organizers who collect registration data via JotForm and want to enhance attendee engagement through personalized communication, AI-driven attendee classification, and networking recommendations. The workflow integrates JotForm registration intake, AI analysis with GPT-4o, conditional routing based on attendee type, personalized email notifications, and logging into a Google Sheets database for analytics.

Logical blocks:

- **1.1 Event Registration Intake:** Receives raw registration data from JotForm and structures it with unique identifiers and timestamps.

- **1.2 AI Attendee Profiling:** Uses an AI agent (LangChain with OpenAI GPT-4o) to analyze attendee data and generate a detailed persona profile, scoring, and recommendations.

- **1.3 AI Response Parsing:** Cleans and parses AI JSON output to merge with registration data, handling parsing errors gracefully.

- **1.4 Smart Routing:** Routes attendees based on their type (VIP, Speaker, Sponsor, First-Time, Standard) to send targeted emails and alerts.

- **1.5 Email Notifications:** Sends personalized emails — VIP alerts and welcomes, first-timer welcomes, and standard registration confirmations.

- **1.6 Analytics Database Logging:** Appends all processed attendee data into a Google Sheets document for tracking and segmentation.

---

### 2. Block-by-Block Analysis

#### 1.1 Event Registration Intake

- **Overview:** Captures raw registration data from JotForm submissions, extracts and normalizes key attendee fields into a structured JSON object. Assigns a unique registration ID and timestamp.

- **Nodes Involved:**  
  - JotForm Trigger  
  - Extract Registration Data (Set node)  
  - Sticky Note (documentation)

- **Node Details:**

  - **JotForm Trigger**  
    - Type: Trigger node for JotForm form submissions.  
    - Configuration: Linked to JotForm form ID `252851017263453`. Receives raw form data on each submission.  
    - Input: External webhook from JotForm.  
    - Output: Raw JSON of form fields.  
    - Edge Cases: Form ID mismatch or connectivity issues with JotForm API.  
    - Credentials: JotForm API OAuth2.  

  - **Extract Registration Data**  
    - Type: Set node used to map and transform raw input fields into normalized properties.  
    - Configuration: Uses conditional expressions to handle missing data (e.g., fallback for full name), sets defaults for optional fields (e.g., expertise defaults to "Not specified", dietary restrictions default to "None"). Generates a unique `registration_id` using current timestamp and random string, and sets a registration ISO date string.  
    - Key Expressions: Multiple JavaScript expressions referencing `$json.rawRequest` fields.  
    - Input: Output from JotForm Trigger.  
    - Output: Structured attendee data with consistent property names.  
    - Edge Cases: Missing form fields, unexpected field formats, expression evaluation errors.  

  - **Sticky Note**  
    - Content: Describes this block as “Event Registration Intake” capturing attendee info including name, company, LinkedIn, goals, and producing a structured data object with unique ID.  

#### 1.2 AI Attendee Profiling

- **Overview:** Uses OpenAI GPT-4o via LangChain conversational agent to analyze the normalized attendee profile. Outputs a JSON describing persona classification, engagement scores, objectives, session recommendations, conversation starters, and tips.

- **Nodes Involved:**  
  - OpenAI Chat Model (LangChain LM Node)  
  - AI Attendee Profiling (LangChain Agent Node)  
  - Sticky Note (documentation)

- **Node Details:**

  - **OpenAI Chat Model**  
    - Type: Language model node using GPT-4.1-mini (OpenAI GPT-4o variant).  
    - Configuration: Model set to "gpt-4.1-mini", no additional options. Uses OpenAI API credentials.  
    - Input: Receives prompt from AI Attendee Profiling node (as a language model sub-node).  
    - Output: Raw AI text response with JSON inside.  
    - Edge Cases: API rate limits, network timeouts, partial or malformed responses.  
    - Credentials: OpenAI API key.

  - **AI Attendee Profiling**  
    - Type: LangChain conversational agent node orchestrating the AI prompt.  
    - Configuration:  
      - Prompt includes attendee profile fields interpolated with expressions, requesting ONLY valid JSON output in specified structure without markdown or code blocks.  
      - System message clearly instructs the AI to return raw JSON.  
      - Output parser enabled to extract JSON from AI response.  
    - Input: Structured registration data from Extract Registration Data.  
    - Output: AI-generated JSON profile and recommendations merged with attendee data.  
    - Edge Cases: AI misunderstanding prompt, incomplete JSON, parser failures.  

  - **Sticky Note**  
    - Content: Describes this block as “AI Attendee Profiling” with persona classification, scoring, recommendations using GPT-4o agent.

#### 1.3 AI Response Parsing

- **Overview:** Cleans and parses the AI response text into valid JSON, merges AI analysis with existing attendee data. Handles parse errors by assigning default profile values and logging error messages.

- **Nodes Involved:**  
  - Parse AI Response (Code node)  

- **Node Details:**

  - **Parse AI Response**  
    - Type: Code node using JavaScript to parse AI output.  
    - Configuration:  
      - Iterates over all input items, extracts AI output text.  
      - Removes markdown code blocks if present.  
      - Uses regex to extract JSON object substring.  
      - Parses JSON safely; on failure, falls back to default persona ("tech_professional") and default scores (50), includes parse error message.  
      - Merges parsed analysis with original registration data.  
    - Input: Output from AI Attendee Profiling.  
    - Output: Merged JSON with attendee data and AI profile.  
    - Edge Cases: Parsing failures, unexpected AI output formats.

#### 1.4 Smart Routing

- **Overview:** Routes attendees to different email notification paths based on their attendee type and first-time status. VIPs/Speakers/Sponsors trigger event team alerts and VIP welcome emails; first-time attendees receive a special welcome; others get standard confirmation.

- **Nodes Involved:**  
  - Is VIP/Speaker/Sponsor? (IF node)  
  - First-Time Attendee? (IF node)  
  - Alert Event Team (VIP) (Gmail node)  
  - Send VIP Welcome Email (Gmail node)  
  - Send First-Timer Welcome (Gmail node)  
  - Send Standard Confirmation (Gmail node)  
  - Sticky Note (documentation)

- **Node Details:**

  - **Is VIP/Speaker/Sponsor?**  
    - Type: IF node evaluating if `attendee_type` equals "VIP", "Speaker", or "Sponsor".  
    - Input: Parsed attendee data with AI profile.  
    - Output: Branches to VIP alert or next IF node.  
    - Edge Cases: Case sensitivity, missing or malformed attendee_type field.

  - **First-Time Attendee?**  
    - Type: IF node checking if `first_time_attendee` equals "Yes".  
    - Input: Output from previous IF node fallback.  
    - Output: Branches to first-timer email or standard confirmation email.  
    - Edge Cases: Field case sensitivity, missing data.

  - **Alert Event Team (VIP)**  
    - Type: Gmail node sending HTML email alert about VIP registration to event team email.  
    - Configuration: Rich HTML email includes attendee details, profile scores, next steps, and registration ID.  
    - Input: VIP branch from IF node.  
    - Credentials: Gmail OAuth2.  
    - Edge Cases: Email delivery failures, invalid email addresses.

  - **Send VIP Welcome Email**  
    - Type: Gmail node sending personalized VIP welcome email to attendee.  
    - Configuration: HTML email describing VIP benefits, event details, and personalized profile info.  
    - Input: Output from Alert Event Team (VIP) node.  
    - Credentials: Gmail OAuth2.  
    - Edge Cases: Email bounces, invalid recipient email.

  - **Send First-Timer Welcome**  
    - Type: Gmail node sending welcome email to first-time attendees.  
    - Configuration: HTML email with first-timer tips, assigned buddy info, conversation starters (top 3 from AI).  
    - Input: IF node branch for first-time attendees.  
    - Credentials: Gmail OAuth2.  
    - Edge Cases: Missing conversation starters, email delivery issues.

  - **Send Standard Confirmation**  
    - Type: Gmail node sending standard registration confirmation email.  
    - Configuration: HTML email with event details, registration ID, and profile persona.  
    - Input: IF node branch for standard attendees.  
    - Credentials: Gmail OAuth2.  
    - Edge Cases: Email delivery failures.

  - **Sticky Note**  
    - Content: Summarizes routing logic: VIP/Speaker/Sponsor → Slack & VIP email alert; First-Time → Buddy & tips; Standard → Confirmation & agenda; all logged to database.

#### 1.5 Analytics Database Logging

- **Overview:** Logs all processed attendee data including AI profiles and routing outcomes to a Google Sheets spreadsheet for analytics and event management.

- **Nodes Involved:**  
  - Log to Event Database (Google Sheets node)  
  - Sticky Note (documentation)

- **Node Details:**

  - **Log to Event Database**  
    - Type: Google Sheets append node.  
    - Configuration: Appends entire input JSON object automatically mapped to sheet columns.  
    - Sheet: Spreadsheet ID `1PqqxQWFZ-R8o58_lscSAGR5GyB24MqH2NKYjKqEVsw8`, sheet `gid=0` ("Event Registration").  
    - Credentials: Google Sheets OAuth2.  
    - Input: Outputs from all email sending nodes (VIP welcome, first-timer welcome, standard confirmation).  
    - Edge Cases: API limits, permission errors, schema mismatches.

  - **Sticky Note**  
    - Content: Notes this block as “Analytics Database” tracking persona, scores, goals, and status for segmentation and ROI insights.

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                        | Input Node(s)                 | Output Node(s)                        | Sticky Note                                                                                       |
|----------------------------|---------------------------------|-------------------------------------|------------------------------|-------------------------------------|-------------------------------------------------------------------------------------------------|
| JotForm Trigger            | JotForm Trigger                 | Receive raw form submissions         | (Trigger)                    | Extract Registration Data            | ## 📝 Event Registration Intake Captures attendee info via JotForm: name, company, title, LinkedIn, interests, goals, networking objectives. **Output:** Structured data with unique ID. |
| Extract Registration Data  | Set                             | Normalize and structure registration data | JotForm Trigger             | AI Attendee Profiling                | See above                                                                                       |
| AI Attendee Profiling      | LangChain Agent (AI node)       | Generate attendee persona and scores | Extract Registration Data    | Parse AI Response                   | ## 🎯 AI Attendee Profiling Classifies persona, scores engagement/influence/value, identifies objectives, recommends sessions. **Tech:** AI Agent with OpenAI GPT-4o             |
| OpenAI Chat Model          | LangChain LM Chat Model         | Underlying AI language model call   | AI Attendee Profiling (lmChat) | AI Attendee Profiling                | See above                                                                                       |
| Parse AI Response          | Code                            | Parse and clean AI JSON response    | AI Attendee Profiling        | Is VIP/Speaker/Sponsor?, First-Time Attendee? |                                                                                                 |
| Is VIP/Speaker/Sponsor?    | IF                              | Check if attendee is VIP, Speaker or Sponsor | Parse AI Response          | Alert Event Team (VIP) / First-Time Attendee? | ## 🚦 Smart Routing VIP/Speaker/Sponsor: Slack → VIP email First-Timer: Buddy assignment → Tips Standard: Confirmation → Agenda All: Log to database                     |
| Alert Event Team (VIP)     | Gmail                           | Send VIP registration alert to event team | Is VIP/Speaker/Sponsor?      | Send VIP Welcome Email              | See above                                                                                       |
| Send VIP Welcome Email     | Gmail                           | Send personalized VIP welcome email | Alert Event Team (VIP)        | Log to Event Database               | See above                                                                                       |
| First-Time Attendee?       | IF                              | Check if attendee is first-time     | Parse AI Response            | Send First-Timer Welcome / Send Standard Confirmation | See above                                                                                       |
| Send First-Timer Welcome   | Gmail                           | Send welcome email with tips to first-timers | First-Time Attendee?         | Log to Event Database               | See above                                                                                       |
| Send Standard Confirmation | Gmail                           | Send standard confirmation email    | First-Time Attendee?         | Log to Event Database               | See above                                                                                       |
| Log to Event Database      | Google Sheets                   | Append attendee data to analytics sheet | Send VIP Welcome Email, Send First-Timer Welcome, Send Standard Confirmation | (End)                              | ## 📊 Analytics Database Tracks registrations: persona, scores, goals, status. **Use Cases:** Segmentation, insights, ROI.                        |
| Sticky Note                | Sticky Note                    | Documentation                      | -                            | -                                   | Multiple notes across workflow as detailed in block descriptions.                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger node:**
   - Type: JotForm Trigger  
   - Configure with your JotForm API credentials.  
   - Select form ID: `252851017263453` (replace with your own).  
   - This node listens for new form submissions.

2. **Add Set node "Extract Registration Data":**
   - Connect output of JotForm Trigger to this node.  
   - Map form fields to standardized properties with expressions:  
     - `attendee_name`: Use form field `q3_fullName` or combine first and last name fields; fallback "Unknown".  
     - Map email, company, job title, LinkedIn URL, industry, interests, event goals, networking goals, expertise (default "Not specified"), looking for, session preferences, dietary restrictions (default "None"), attendee type (default "Standard"), first-time attendee (default "No").  
     - Generate `registration_id` as `EVT-<timestamp>-<randomstring>`.  
     - Set `registration_date` as current ISO string.  
   - No credentials needed.

3. **Add LangChain OpenAI Chat Model node:**
   - Type: LangChain LM Chat Model.  
   - Connect output of "Extract Registration Data" to the AI Attendee Profiling (agent) node (see next).  
   - Configure model as "gpt-4.1-mini" or GPT-4 equivalent.  
   - Provide OpenAI API credentials.

4. **Add LangChain Agent node "AI Attendee Profiling":**
   - Connect output of "Extract Registration Data" to this node.  
   - Use the LangChain agent configured with conversationalAgent.  
   - Set prompt text embedding attendee data fields with expressions.  
   - Add a system message instructing to return ONLY valid JSON (no markdown/code blocks).  
   - Enable output parser to parse returned JSON.  
   - Set the LangChain LM Chat Model node as its language model (connect via "ai_languageModel" input).

5. **Add Code node "Parse AI Response":**
   - Connect output of "AI Attendee Profiling" to this node.  
   - Paste JavaScript code that:  
     - Iterates input items, extracts AI text output.  
     - Removes any markdown code blocks.  
     - Extracts JSON substring using regex.  
     - Parses JSON safely, falls back to defaults on error.  
     - Merges AI data with original registration data.  
   - No credentials needed.

6. **Add IF node "Is VIP/Speaker/Sponsor?":**
   - Connect output of "Parse AI Response" to this IF node.  
   - Configure condition: attendee_type equals "VIP" OR "Speaker" OR "Sponsor" (case sensitive).  
   - Set two output branches: true (VIP path), false (next check).

7. **Add IF node "First-Time Attendee?":**
   - Connect false output of previous IF node to here.  
   - Configure condition: first_time_attendee equals "Yes" (case sensitive).  
   - Two outputs: true (first-time welcome), false (standard confirmation).

8. **Add Gmail node "Alert Event Team (VIP)":**
   - Connect true output of "Is VIP/Speaker/Sponsor?" to this node.  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient to event team email (e.g., akshitgupta29@gmail.com).  
   - Compose rich HTML email with VIP details, profile scores, next steps, registration ID.  
   - Subject example: "🌟 VIP Registration Alert - [attendee_name]".

9. **Add Gmail node "Send VIP Welcome Email":**
   - Connect output of "Alert Event Team (VIP)" node here.  
   - Configure Gmail credentials.  
   - Send personalized VIP welcome email to attendee’s email address.  
   - Include VIP benefits, event details, persona, registration ID, and contact info.

10. **Add Gmail node "Send First-Timer Welcome":**
    - Connect true output of "First-Time Attendee?" IF node here.  
    - Configure Gmail credentials.  
    - Compose welcome email with first-timer tips, assigned buddy info (e.g., Alex Johnson), top 3 conversation starters from AI data, registration ID.

11. **Add Gmail node "Send Standard Confirmation":**
    - Connect false output of "First-Time Attendee?" IF node here.  
    - Configure Gmail credentials.  
    - Compose standard registration confirmation email with event dates, location, registration ID, and persona.

12. **Add Google Sheets node "Log to Event Database":**
    - Connect outputs from:  
      - "Send VIP Welcome Email"  
      - "Send First-Timer Welcome"  
      - "Send Standard Confirmation"  
    - Configure with Google Sheets OAuth2 credentials.  
    - Set document ID to your event registration sheet.  
    - Set operation to Append.  
    - Use automatic mapping to columns for all input JSON fields.  
    - Sheet name or GID: use first sheet or specify as needed.

13. **Add Sticky Notes as needed for documentation:**
    - Create notes describing each major block: Intake, AI Profiling, Smart Routing, Analytics.  
    - Position and color for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow uses OpenAI GPT-4o via LangChain for AI attendee profiling with strict JSON output format instructions.      | AI integration best practice for clean parsing and avoiding markdown artifacts.                  |
| Personalized emails use rich HTML templates styled inline for consistent rendering across email clients.              | Email styling optimized for Gmail and common clients.                                           |
| Unique registration ID combines timestamp and random string for traceability.                                         | Useful for cross-referencing logs and communications.                                           |
| Google Sheets serves as the backend analytics database enabling segmentation and ROI tracking on event registrations. | Spreadsheet ID: 1PqqxQWFZ-R8o58_lscSAGR5GyB24MqH2NKYjKqEVsw8                                     |
| JotForm form ID is specific to the deployed form; replace with your own form ID.                                     | Form ID: 252851017263453                                                                         |
| Use OAuth2 credentials for Gmail and Google Sheets to ensure secure and authorized API interactions.                  | Credentials named “Gmail account - Deepanshi” and “Google Sheets account - Deepanshi” in example |
| VIP, Speaker, Sponsor identification drives special alert emails and concierge workflows.                            | Can be expanded to integrate Slack or CRM notifications.                                        |

---

This completes the detailed, structured reference documentation for the "Event Registration & AI Networking Matchmaker with GPT-4o, JotForm & Google Sheets" workflow. It enables advanced users and automation agents to fully understand, reproduce, and maintain the workflow effectively.