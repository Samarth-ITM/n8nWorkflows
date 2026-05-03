Qualify & Reach Out to B2B Leads with Groq AI, Apollo, Gmail & Sheets

https://n8nworkflows.xyz/workflows/qualify---reach-out-to-b2b-leads-with-groq-ai--apollo--gmail---sheets-5832


# Qualify & Reach Out to B2B Leads with Groq AI, Apollo, Gmail & Sheets

---

## 1. Workflow Overview

This workflow automates the qualification and outreach process for B2B leads using AI and various integrated services. It targets businesses that collect lead data via a form or Google Sheets and want to enrich, score, and engage those leads automatically through email outreach and record-keeping.

The workflow is divided into the following logical blocks:

- **1.1 Input Reception:** Captures lead data from either a form submission or a Google Sheets trigger.
- **1.2 Lead Enrichment:** Uses the Apollo API to match and enrich the lead data with professional details.
- **1.3 Lead Qualification:** Evaluates the lead’s industry and job title using an AI scoring model to assess potential.
- **1.4 Conditional Branching:** Determines if the lead qualifies based on the AI score.
- **1.5 Outreach Composition and Sending:** Uses an AI agent to generate a personalized outreach email and sends it via Gmail.
- **1.6 Record Keeping:** Appends the lead data and qualification results to a Google Sheet for tracking.

---

## 2. Block-by-Block Analysis

### 2.1 Input Reception

**Overview:**  
This block captures incoming lead data either from an online form submission or from a new row added in a Google Sheet. It provides the initial data set for processing.

**Nodes Involved:**  
- On form submission (Form Trigger)  
- Google Sheets Trigger (not actively used here, but present)  
- Webhook (present but not connected in the main path)

**Node Details:**

- **On form submission**  
  - *Type:* Form Trigger (Webhook-based)  
  - *Role:* Captures form input fields: Full Name, Email, Phone Number, LinkedIn URL.  
  - *Configuration:* Form titled "Contact us" with required fields for name, email, phone, LinkedIn URL.  
  - *Inputs:* External form submission via webhook.  
  - *Outputs:* JSON containing form field data.  
  - *Edge Cases:* Invalid or incomplete form inputs; submission delays; webhook connectivity errors.

- **Google Sheets Trigger**  
  - *Type:* Google Sheets Trigger  
  - *Role:* Intended to trigger workflow when a new row is added to a specified sheet.  
  - *Configuration:* Polls every minute; documentId and sheetName are empty (not fully configured).  
  - *Inputs:* New row event in Google Sheets.  
  - *Outputs:* JSON row data.  
  - *Edge Cases:* Misconfiguration of document ID or sheet name; API quota limits; network timeouts.

- **Webhook**  
  - *Type:* Webhook  
  - *Role:* Alternative entry point (not connected downstream in this workflow).  
  - *Configuration:* Path defined but no downstream connections.  
  - *Edge Cases:* Unused node; no effect on workflow unless connected.

---

### 2.2 Lead Enrichment

**Overview:**  
Using the Apollo API, this block enriches the lead by matching submitted data to detailed professional profiles, adding information like job title, organization, and employment history.

**Nodes Involved:**  
- HTTP Request (Apollo People Match)

**Node Details:**

- **HTTP Request**  
  - *Type:* HTTP Request (POST)  
  - *Role:* Calls Apollo API `/people/match` endpoint to match lead data.  
  - *Configuration:* Sends name, email, phone, and LinkedIn URL as query parameters. Uses HTTP Header Authentication for API key.  
  - *Inputs:* JSON from form submission or Google Sheets trigger.  
  - *Outputs:* JSON response including detailed person object with fields like name, email, title, employment history, organization details.  
  - *Expressions:* Dynamic query parameters extracted from the input JSON fields.  
  - *Edge Cases:* API authentication errors, rate limits, data mismatch, incomplete or missing input fields, network errors.

---

### 2.3 Lead Qualification

**Overview:**  
This block uses an AI agent powered by LangChain and Groq’s language model to score the lead on two criteria: industry alignment and job title relevance. The score ranges from 1 to 10.

**Nodes Involved:**  
- AI Agent1 (Lead Scoring)  
- Groq Chat Model1 (Language Model for AI Agent1)  
- If (Conditional check on score)

**Node Details:**

- **Groq Chat Model1**  
  - *Type:* LangChain Chat Model (Groq)  
  - *Role:* Provides language model capabilities to the AI Agent1.  
  - *Configuration:* Uses Groq API credentials. No additional options set.  
  - *Edge Cases:* API limits, credentials errors, response timeouts.

- **AI Agent1**  
  - *Type:* LangChain Agent  
  - *Role:* Receives lead data and prompts the model to score lead potential based on industry and job title rules.  
  - *Configuration:* Prompt explicitly defines scoring criteria and expects a numeric output 1-10 without explanation.  
  - *Inputs:* JSON person object from Apollo HTTP Request.  
  - *Outputs:* Numeric score as string or number.  
  - *Expressions:* Uses lead details dynamically inserted into prompt.  
  - *Edge Cases:* Malformed prompt outputs, unexpected AI responses, parsing errors.

- **If**  
  - *Type:* Conditional node  
  - *Role:* Checks if the AI score is greater than or equal to 6 to proceed with outreach.  
  - *Configuration:* Condition: `Number($json.output) >= 6`  
  - *Inputs:* Output from AI Agent1.  
  - *Outputs:* Routes true (qualified lead) or false (not qualified).  
  - *Edge Cases:* Non-numeric or missing score value.

---

### 2.4 Outreach Composition and Sending

**Overview:**  
This block composes a personalized outreach email using an AI agent, then sends it via Gmail to the qualified lead.

**Nodes Involved:**  
- AI Agent (Email Composition)  
- Groq Chat Model (Language Model for AI Agent)  
- Structured Output Parser  
- Send a message in Gmail

**Node Details:**

- **Groq Chat Model**  
  - *Type:* LangChain Chat Model (Groq)  
  - *Role:* Language model for AI Agent to generate email content.  
  - *Configuration:* Groq API credentials.  
  - *Edge Cases:* API errors, timeouts.

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Generates a short, professional outreach email to schedule a call, using enriched lead info dynamically.  
  - *Configuration:* Prompt instructs tone, message format, placeholders for lead name, organization, and email. Output is structured as an email to send.  
  - *Inputs:* JSON person details from Apollo HTTP Request.  
  - *Outputs:* Structured email content with To, Subject, and Message fields.  
  - *Edge Cases:* Unexpected AI output format, missing input data.

- **Structured Output Parser**  
  - *Type:* LangChain Structured Output Parser  
  - *Role:* Parses AI Agent output into structured fields for further use.  
  - *Inputs:* Raw AI Agent output.  
  - *Outputs:* Parsed fields for email sending.  
  - *Edge Cases:* Parsing failures if AI output deviates.

- **Send a message in Gmail**  
  - *Type:* Gmail Tool  
  - *Role:* Sends the composed email to the lead’s email address.  
  - *Configuration:* Uses OAuth2 Gmail credentials. Email content and recipient dynamically set from AI Agent output and Apollo data.  
  - *Inputs:* To address and message content from AI Agent.  
  - *Edge Cases:* Gmail API authorization errors, quota limits, invalid email addresses.

---

### 2.5 Record Keeping

**Overview:**  
This block logs all the lead information along with the qualification score into a Google Sheet for tracking and analysis.

**Nodes Involved:**  
- Append row in sheet (Google Sheets node)

**Node Details:**

- **Append row in sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Appends a new row to a specified Google Sheet with lead data and score.  
  - *Configuration:* Maps form fields, Apollo enrichment data, and AI score into specific columns (Name, Email, Phone, Job Title, Organization, LinkedIn URL, Lead Potential).  
  - *Inputs:* Data from form submission and HTTP Request (Apollo), plus AI Agent output.  
  - *Credentials:* Google Sheets OAuth2 credentials.  
  - *Edge Cases:* API limits, sheet access permissions, mismatched schemas.

---

## 3. Summary Table

| Node Name              | Node Type                      | Functional Role                | Input Node(s)             | Output Node(s)           | Sticky Note                              |
|------------------------|--------------------------------|-------------------------------|---------------------------|--------------------------|----------------------------------------|
| On form submission     | Form Trigger                   | Captures lead data from form   | —                         | HTTP Request             |                                        |
| HTTP Request           | HTTP Request                  | Enrich lead via Apollo API     | On form submission, Google Sheets Trigger | AI Agent1               |                                        |
| AI Agent1              | LangChain Agent               | Scores lead potential          | HTTP Request              | If                       |                                        |
| Groq Chat Model1       | LangChain Chat Model (Groq)   | Provides AI model for scoring  | —                         | AI Agent1                |                                        |
| If                     | Conditional                  | Checks lead score threshold    | AI Agent1                 | AI Agent                 |                                        |
| AI Agent               | LangChain Agent               | Composes outreach email        | If (true)                 | Append row in sheet, Send a message in Gmail |                                        |
| Groq Chat Model        | LangChain Chat Model (Groq)   | Provides AI model for email    | —                         | AI Agent                 |                                        |
| Structured Output Parser | LangChain Output Parser      | Parses AI email output         | AI Agent                  | Send a message in Gmail  |                                        |
| Send a message in Gmail | Gmail Tool                   | Sends outreach email           | Structured Output Parser  | AI Agent (via ai_tool)   |                                        |
| Append row in sheet    | Google Sheets Append Row       | Logs lead data and score       | AI Agent                  | —                        |                                        |
| Google Sheets Trigger  | Google Sheets Trigger          | Alternative input trigger      | —                         | HTTP Request             | Not configured (empty documentId/sheetName) |
| Webhook                | Webhook                       | Alternative input trigger (unused) | —                      | HTTP Request (not connected) | Not used in main flow                   |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node:**  
   - Type: Form Trigger  
   - Configure form titled "Contact us" with these required fields:  
     - "What's your Full Name?" (text)  
     - "What is your email address?" (email)  
     - "What's your phone number?" (number)  
     - "What's your Linkedin Profile URL" (text)  
   - Set form description to "get back to you".  
   - This node will serve as the initial data capture point.

2. **Add an HTTP Request node to enrich lead data:**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.apollo.io/api/v1/people/match`  
   - Use HTTP Header Authentication with Apollo API key credentials.  
   - Pass query parameters dynamically using expressions:  
     - `name` → `{{$json["What's your Full Name?"]}}`  
     - `email` → `{{$json["What is your email address?"]}}`  
     - `reveal_phone_number` → `{{$json["What's your phone number?"]}}`  
     - `linkedin_url` → `{{$json["What's your Linkedin Profile URL?"]}}`  
   - Connect the Form Trigger node’s output to this HTTP Request node.

3. **Add Groq Chat Model1 node:**  
   - Type: LangChain Chat Model (Groq)  
   - Configure with Groq API credentials.

4. **Add AI Agent1 node for lead scoring:**  
   - Type: LangChain Agent  
   - Assign Groq Chat Model1 as the language model.  
   - Use this prompt (with expressions) to score lead potential from 1-10:  
     ```
     You are an expert lead qualifier.

     Score the lead from 1 to 10 based on these two criteria:

     — INDUSTRY MATCH (5 points max) —
     - 5 points: Industry is AI, Artificial Intelligence, Machine Learning, Data Science
     - 3-4 points: Industry is closely related (e.g., analytics, automation, enterprise AI tools)
     - 0-2 points: Unrelated industry

     — JOB TITLE MATCH (5 points max) —
     - 5 points: Decision-makers or influencers in AI (e.g., Head of AI, AI Director, CTO, AI Product Manager)
     - 3-4 points: Mid-level AI professionals (e.g., ML Engineer, Data Scientist)
     - 0-2 points: Not related to AI or junior roles

     Return only a single digit between 1 and 10 — no explanation or extra text.

     Lead details:
     Name: {{ $json.person.name }}
     Job Title: {{ $json.person.title }}
     Organization: {{ $json.person.employment_history[0].organization_name }}
     ```
   - Connect HTTP Request node output to AI Agent1 input.

5. **Add an If node to check score threshold:**  
   - Condition: Number($json.output) >= 6  
   - Connect AI Agent1 output to this If node.

6. **Add Groq Chat Model node for email composition:**  
   - Type: LangChain Chat Model (Groq)  
   - Use Groq API credentials.

7. **Add AI Agent node for email generation:**  
   - Type: LangChain Agent  
   - Assign Groq Chat Model as language model.  
   - Use this prompt template with expressions (instruct AI to compose a polite outreach email):  
     ```
     You are a professional B2B sales assistant.

     Compose a short, professional, and polite outreach email to schedule a call with the following lead.

     Tone: concise and respectful. The goal is to open a conversation and set up a meeting.

     Use this format:
     ---
     Send a message in Gmail:
     To: {{ $('HTTP Request').item.json.person.email }}
     Subject: Exploring Collaboration in AI Solutions

     Hi {{ $('HTTP Request').item.json.person.name }},

     I hope this message finds you well.

     I’m reaching out from ABC Pvt Ltd—we specialize in tailored IT solutions, including AI and machine learning services. Given your role at {{ $('HTTP Request').item.json.person.employment_history[0].organization_name }}, I’d love to briefly connect and explore potential synergies.

     Would you be available for a quick call sometime this week?

     Best regards,  
     ABC XYZ  
     ABC Pvt Ltd
     ---
     Here is the lead detail:
     Name: {{ $('HTTP Request').item.json.person.name }}
     Email: {{ $('HTTP Request').item.json.person.email }}
     Job Title: {{ $('HTTP Request').item.json.person.title }}
     Organization: {{ $('HTTP Request').item.json.person.employment_history[0].organization_name }}

     Instructions:
     Tools - Send a message in Gmail Tools for send mails
     ```
   - Connect If node’s true output to this AI Agent node.

8. **Add Structured Output Parser node:**  
   - Type: LangChain Structured Output Parser  
   - Connect AI Agent output to this node to parse email content.

9. **Add Send a message in Gmail node:**  
   - Type: Gmail Tool  
   - Configure with Gmail OAuth2 credentials.  
   - Set "Send To" to `={{ $('HTTP Request').item.json.person.email }}`  
   - Set "Subject" and "Message" dynamically from AI agent's parsed output.  
   - Connect Structured Output Parser output to this Gmail node.

10. **Add Append row in sheet node:**  
    - Type: Google Sheets Append Row  
    - Configure Google Sheets OAuth2 credentials.  
    - Set document ID and sheet name to your target sheet.  
    - Map columns:  
      - Name → `={{ $('On form submission').item.json["What's your Full Name?"] }}`  
      - Email → `={{ $('On form submission').item.json["What is your email address?"] }}`  
      - Phone → `={{ $('On form submission').item.json["What's your phone number?"] }}`  
      - Job Title → `={{ $('HTTP Request').item.json.person.title }}`  
      - Organization → `={{ $('HTTP Request').item.json.person.employment_history[0].organization_name }}`  
      - Linkedin URL → `={{ $('On form submission').item.json["What's your Linkedin Profile URL"] }}`  
      - Lead Potential → `={{ $json.output }}` (score from AI Agent1)  
    - Connect AI Agent (email composition) output to this node.

11. **Link nodes in order:**  
    - On form submission → HTTP Request → AI Agent1 → If → (true branch) AI Agent → Structured Output Parser → Send a message in Gmail  
    - AI Agent → Append row in sheet

12. **Test with real credentials and API keys:**  
    - Apollo API key for HTTP Request node  
    - Groq API key for Groq Chat Model nodes  
    - Gmail OAuth2 credentials for Gmail node  
    - Google Sheets OAuth2 credentials for Append row node

---

## 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The workflow uses Apollo API for lead enrichment; ensure you have a valid Apollo account and API key.  | https://apollo.io/api/                                                                           |
| Groq AI is used as the language model provider; requires Groq API credentials for LangChain nodes.     | https://groq.com/                                                                               |
| Gmail OAuth2 setup is required for sending emails; configure OAuth2 credentials in n8n settings.       | https://developers.google.com/gmail/api/auth/about-auth                                         |
| Google Sheets OAuth2 setup is required for appending rows; ensure the sheet has correct permissions.   | https://developers.google.com/sheets/api/guides/authorizing                                      |
| Prompt design in AI Agents is critical for consistent and expected outputs; maintain prompt clarity.   |                                                                                                 |
| For large scale use, consider handling API rate limits and errors with retries or error workflows.    |                                                                                                 |

---

**Disclaimer:**  
The text provided comes exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled are legal and public.

---