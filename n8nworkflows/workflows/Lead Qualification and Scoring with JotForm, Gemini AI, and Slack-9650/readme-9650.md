Lead Qualification and Scoring with JotForm, Gemini AI, and Slack

https://n8nworkflows.xyz/workflows/lead-qualification-and-scoring-with-jotform--gemini-ai--and-slack-9650


# Lead Qualification and Scoring with JotForm, Gemini AI, and Slack

### 1. Workflow Overview

This workflow automates the lead qualification and scoring process for sales inquiries submitted via a JotForm form. It employs Google Gemini AI to analyze the lead message, detect spam, and assign a lead score. Based on the score, it routes leads into categories ("Hot," "Warm," or "Cold") and sends personalized follow-up emails accordingly. Additionally, it sends a Slack notification for valid leads to alert the sales team immediately.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures new JotForm submissions and initiates processing.
- **1.2 AI Processing:** Uses Google Gemini AI to analyze the lead message for spam and lead quality scoring.
- **1.3 Spam Filtering:** Stops processing if the submission is identified as spam.
- **1.4 Sales Team Notification:** Sends a Slack alert for valid leads.
- **1.5 Lead Categorization:** Classifies the lead as Hot, Warm, or Cold based on score.
- **1.6 Personalized Email Response:** Sends a customized email based on lead category.
- **1.7 No-Op Handling:** Handles spam leads by halting workflow without action.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block starts the workflow with a trigger node that listens for new form submissions on a specific JotForm. It collects key lead information such as full name, email, role, company size, and the inquiry message.

- **Nodes Involved:**  
  - JotForm Trigger

- **Node Details:**

  - **JotForm Trigger**  
    - Type: Trigger node for JotForm submissions  
    - Configuration: Linked to a specific form (ID: 252856078065061) with JotForm API credentials  
    - Key Expressions: None (trigger event-based)  
    - Input Connections: None (starting node)  
    - Output Connections: Feeds into AI Agent node  
    - Version Requirements: n8n standard JotForm node, no special version needed  
    - Potential Failures: API authentication error, webhook setup failure, form ID mismatch  
    - Sticky Note: Explains required form fields and setup for JotForm (Full Name, Work Email, Role, Number of Employees, "How can we help you?" text area) with link to JotForm signup. Advises updating field names if customized.  

#### 1.2 AI Processing

- **Overview:**  
  This block analyzes the lead's inquiry text using Google Gemini AI via LangChain integration. It checks if the message is spam and assigns a lead quality score from 1 to 10. The output is parsed into structured JSON.

- **Nodes Involved:**  
  - AI Agent  
  - Google Gemini Chat Model (language model node)  
  - Structured Output Parser

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: Language model node using Google Gemini (PaLM) API  
    - Configuration: Uses Google Palm API credentials  
    - Input: Receives prompt text from AI Agent node  
    - Output: Provides raw AI model response  
    - Failures: API rate limits, authentication errors, network timeouts  

  - **AI Agent**  
    - Type: LangChain Agent node (version 2.2)  
    - Configuration:  
      - Prompt instructs AI to analyze the message field `"How can we help?"` for spam detection and lead scoring  
      - Output expected strictly as JSON with keys `is_spam` (boolean) and `lead_score` (integer 1-10)  
      - Uses Google Gemini Chat Model as backend  
    - Key Expressions: `={{ $json['How can we help?'] }}` for injecting message content dynamically  
    - Input: From JotForm Trigger  
    - Output: Passes structured JSON to If Spam node  
    - Version Specific: Uses output parser integration for structured JSON  
    - Potential Failures: Expression errors if field names don’t match; AI response parsing errors; API failures  
    - Sticky Note: Describes the AI’s role and reminds user to connect Google AI credentials and verify field names  

  - **Structured Output Parser**  
    - Type: LangChain structured output parser node  
    - Configuration: Defines expected JSON schema example to validate AI output  
    - Input: Receives raw AI response  
    - Output: Parsed JSON with keys `is_spam` and `lead_score`  
    - Failures: Parsing errors if AI returns invalid JSON  

#### 1.3 Spam Filtering

- **Overview:**  
  This block filters out spam leads by checking the `is_spam` boolean from the AI output. Spam submissions are discarded to avoid wasting sales team resources.

- **Nodes Involved:**  
  - If Spam  
  - Nothing to do for spams! (NoOp)  
  - Send lead alert to sales (Slack notification)

- **Node Details:**

  - **If Spam**  
    - Type: Conditional (If) node  
    - Configuration: Checks if `is_spam` equals `true`  
    - Input: From AI Agent output  
    - Output:  
      - If `true`: routes to NoOp node (halts further action)  
      - If `false`: routes to Slack notification node  
    - Potential Failures: Expression errors if AI output malformed  

  - **Nothing to do for spams!**  
    - Type: NoOp (no operation) node  
    - Configuration: None  
    - Input: From If Spam (spam condition)  
    - Output: None (stops workflow)  
    - Purpose: Silently discards spam leads  

  - **Send lead alert to sales**  
    - Type: Slack node  
    - Configuration:  
      - Sends formatted message to a designated Slack channel (`#sales-leads` or channel ID `C09M7DZFLQY`)  
      - Message includes lead name, company size, inquiry, and lead score  
      - Uses Slack API credentials  
    - Input: From If Spam (non-spam path)  
    - Output: Forwards to Lead Categorization (Switch) node  
    - Failures: Slack API authentication issues, channel permission errors  
    - Sticky Note: Advises connecting Slack credentials and selecting proper channel  

#### 1.4 Lead Categorization

- **Overview:**  
  This block classifies the lead into Hot, Warm, or Cold categories based on the AI-generated lead score to determine the appropriate email response.

- **Nodes Involved:**  
  - Switch

- **Node Details:**

  - **Switch**  
    - Type: Switch node (version 3.3)  
    - Configuration:  
      - Routes leads based on `lead_score` from AI Agent output:  
        - Hot Lead: score ≥ 8  
        - Warm Lead: score ≥ 5 and < 8  
        - Cold Lead: score ≤ 4  
    - Input: From Slack notification node  
    - Output: Connects to corresponding email reply nodes  
    - Potential Failures: Expression errors if AI data missing or malformed  
    - Sticky Note: Explains scoring rules and encourages customization  

#### 1.5 Personalized Email Response

- **Overview:**  
  Sends a customized follow-up email to the lead based on their category. Hot leads receive a prompt to book a meeting; warm and cold leads get tailored messages appropriate to their interest level.

- **Nodes Involved:**  
  - Hot Reply (Gmail node)  
  - Warm Reply (Gmail node)  
  - Cold Reply (Gmail node)

- **Node Details:**

  - **Hot Reply**  
    - Type: Gmail node (send email)  
    - Configuration:  
      - Sends to lead’s Work Email  
      - Subject: “Following up on your request”  
      - Message: Personalized, encourages booking a calendar meeting with placeholder link  
      - Uses Gmail OAuth2 credentials  
    - Input: From Switch node (Hot Lead path)  
    - Output: Terminal  
    - Failures: Gmail authentication, invalid email address  
    - Sticky Note: Reminds to customize emails and connect Gmail credentials  

  - **Warm Reply**  
    - Type: Gmail node (send email)  
    - Configuration:  
      - Sends to lead’s Work Email  
      - Subject: “Thanks for contacting us”  
      - Message: Provides case study link and promises response within one business day  
      - Uses Gmail OAuth2 credentials  
    - Input: From Switch node (Warm Lead path)  
    - Output: Terminal  
    - Failures: Same as Hot Reply  

  - **Cold Reply**  
    - Type: Gmail node (send email)  
    - Configuration:  
      - Sends to lead’s Work Email  
      - Subject: “We've received your message”  
      - Message: General acknowledgment without urgency  
      - Uses Gmail OAuth2 credentials  
    - Input: From Switch node (Cold Lead path)  
    - Output: Terminal  
    - Failures: Same as above  

  - Sticky Note: Emphasizes importance of customizing each email and updating placeholder URLs  

---

### 3. Summary Table

| Node Name                 | Node Type                          | Functional Role                  | Input Node(s)            | Output Node(s)               | Sticky Note                                                                                                                                                              |
|---------------------------|----------------------------------|--------------------------------|--------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| JotForm Trigger           | JotForm Trigger                  | Capture form submissions        | None                     | AI Agent                    | Explains required JotForm fields and setup, with link to JotForm signup                                                                                               |
| AI Agent                  | LangChain Agent                  | Analyze message for spam & score| JotForm Trigger          | If Spam                     | Describes AI role, prompt, credentials, and to verify field names                                                                                                     |
| Google Gemini Chat Model  | LangChain LM Chat Google Gemini | Provides AI language model      | AI Agent (as LM)         | AI Agent                    |                                                                                                                                                                        |
| Structured Output Parser  | LangChain Output Parser Structured| Parses AI response to JSON      | AI Agent (as parser)     | AI Agent                    |                                                                                                                                                                        |
| If Spam                   | If                              | Filter spam leads               | AI Agent                 | Nothing to do for spams!, Send lead alert to sales | Explains spam filtering step                                                                                                                                          |
| Nothing to do for spams!  | NoOp                            | Stops spam leads                | If Spam (true)           | None                        |                                                                                                                                                                        |
| Send lead alert to sales  | Slack                           | Notify sales team of valid leads| If Spam (false)          | Switch                      | Advises connecting Slack credentials and choosing channel                                                                                                             |
| Switch                    | Switch                          | Categorize leads by score       | Send lead alert to sales | Hot Reply, Warm Reply, Cold Reply | Explains scoring rules and customization                                                                                                                              |
| Hot Reply                 | Gmail                           | Send email for hot leads        | Switch (Hot Lead)        | None                        | Emphasizes email customization; connect Gmail credentials                                                                                                            |
| Warm Reply                | Gmail                           | Send email for warm leads       | Switch (Warm Lead)       | None                        | Same as Hot Reply                                                                                                                                                      |
| Cold Reply                | Gmail                           | Send email for cold leads       | Switch (Cold Lead)       | None                        | Same as Hot Reply                                                                                                                                                      |
| Sticky Note               | Sticky Note                     | Documentation                  | None                     | None                        | Multiple sticky notes throughout workflow provide step explanations and setup instructions                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create JotForm Trigger node:**  
   - Type: JotForm Trigger  
   - Configure with your JotForm API credentials  
   - Select your form (ensure it includes fields: Full Name, Work Email, Role, Number of Employees, How can we help?)  
   - Position this node as the workflow start  

2. **Add Google Gemini Chat Model node:**  
   - Type: LangChain LM Chat Google Gemini  
   - Set up Google Palm API credentials  
   - Keep default options  

3. **Add Structured Output Parser node:**  
   - Type: LangChain Output Parser Structured  
   - Provide JSON schema example:  
     ```json
     {
       "is_spam": false,
       "lead_score": 9
     }
     ```  

4. **Add AI Agent node:**  
   - Type: LangChain Agent (version 2.2)  
   - Configure prompt:  
     ```
     You are an expert lead qualification assistant. Analyze the form submission message below to check for spam and calculate a lead score.

     Analyze this data:
     - Message: {{ $json['How can we help?'] }}

     Provide your response ONLY in a valid JSON format with two keys:
     1. "is_spam": A boolean (true or false).
     2. "lead_score": An integer from 1 to 10, where 10 is a high-quality lead with a clear, specific request.

     Do not include any text outside of the JSON object.
     ```  
   - Connect Google Gemini Chat Model as the language model backend  
   - Set Structured Output Parser as the output parser  
   - Connect input from JotForm Trigger node  

5. **Add If Spam node:**  
   - Type: If (version 2.2)  
   - Condition: Check if `={{ $json.output.is_spam }}` equals `true` (boolean)  

6. **Add NoOp node ("Nothing to do for spams!"):**  
   - Type: NoOp  
   - Connect to If Spam node’s true output  

7. **Add Slack node ("Send lead alert to sales"):**  
   - Type: Slack  
   - Connect to If Spam node’s false output  
   - Configure Slack API credentials  
   - Select target sales channel (e.g., `#sales-leads`)  
   - Message text example:  
     ```
     🚀 *New Lead Qualified! (Score: {{ $json.output.lead_score }} /10)*

     *Name:* {{ $('JotForm Trigger').item.json['Full Name'] }}
     *Company Size:* {{ $('JotForm Trigger').item.json['Number of Employees'] }}
     *Inquiry:* {{ $('JotForm Trigger').item.json['How can we help?'] }}
     ```  

8. **Add Switch node:**  
   - Type: Switch (version 3.3)  
   - Input: From Slack notification node  
   - Rules:  
     - Hot Lead: `lead_score >= 8`  
     - Warm Lead: `lead_score >= 5`  
     - Cold Lead: `lead_score <= 4`  

9. **Add Gmail nodes for Hot, Warm, and Cold replies:**  
   - Type: Gmail (send email)  
   - Configure Gmail OAuth2 credentials  
   - Messages:  
     - **Hot Reply:**  
       Subject: "Following up on your request"  
       Body: Personalized message encouraging booking a meeting, include calendar link placeholder  
     - **Warm Reply:**  
       Subject: "Thanks for contacting us"  
       Body: Message with case study link and promise to respond soon  
     - **Cold Reply:**  
       Subject: "We've received your message"  
       Body: General acknowledgment  
   - Connect each Gmail node to the corresponding Switch node output  

10. **Verify all expressions:**  
    - Ensure all field names in expressions match your JotForm field names exactly  
    - Adjust message templates and placeholder links to your company’s branding and URLs  

11. **Test the workflow:**  
    - Submit test data through your JotForm form to verify end-to-end processing  
    - Check Slack notifications and emails for correctness  

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Recommended to customize all email templates with your company’s voice and replace placeholder URLs such as booking links.  | Step 6 Sticky Note in workflow                                                                   |
| Required JotForm fields: Full Name, Work Email, Role, Number of Employees, and "How can we help you?" text area.              | Step 1 Sticky Note with form setup instructions                                                   |
| Connect Google AI credentials for Gemini model to enable AI analysis.                                                        | Step 2 Sticky Note                                                                                |
| Connect Slack credentials and select your sales notification channel.                                                        | Step 4 Sticky Note                                                                                |
| Adjust lead scoring thresholds in the Switch node to fit your sales criteria.                                                | Step 5 Sticky Note                                                                                |
| Use official JotForm signup link: https://www.jotform.com/?partner=atakhalighi                                              | Step 1 Sticky Note                                                                                |

---

**Disclaimer:** The provided content originates exclusively from an automated workflow created in n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly available.