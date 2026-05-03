Transform Hotel Guest Feedback with GPT-4 Sentiment Analysis & Service Recovery

https://n8nworkflows.xyz/workflows/transform-hotel-guest-feedback-with-gpt-4-sentiment-analysis---service-recovery-9811


# Transform Hotel Guest Feedback with GPT-4 Sentiment Analysis & Service Recovery

### 1. Workflow Overview

This workflow automates the reception, analysis, and follow-up of hotel guest feedback collected via Jotform. Its purpose is to transform raw guest feedback into actionable insights using GPT-4 powered sentiment analysis, enabling hotel staff to prioritize urgent issues, deliver personalized recovery offers, and foster positive guest relationships. The workflow is designed to reduce negative reviews by automating service recovery and enhancing guest engagement.

Logical blocks included:

- **1.1 Input Reception:** Captures guest feedback submissions from Jotform and extracts key data fields.
- **1.2 AI Sentiment Analysis:** Uses GPT-4 to analyze feedback sentiment, urgency, and recommended actions.
- **1.3 Urgency Routing:** Detects critical/high urgency feedback to alert hotel management via email and Slack.
- **1.4 Service Recovery:** Generates personalized recovery offers for negative feedback or low ratings, sends emails, and creates PMS tickets.
- **1.5 Positive Feedback Handling:** Sends thank-you emails and encourages positive reviews for satisfied guests.
- **1.6 Analytics & Logging:** Logs all feedback and analysis results into Google Sheets for tracking and insights.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Receives guest feedback submissions via Jotform webhook trigger, extracts relevant form fields into structured variables for downstream processing.

**Nodes Involved:**  
- Jotorm Trigger  
- Extract Feedback Data  

**Node Details:**  

- **Jotorm Trigger**  
  - Type: JotForm Trigger  
  - Role: Listens for new form submissions on a specific Jotform ID.  
  - Configuration: Uses webhook with form ID `252862984356471`.  
  - Inputs: Incoming HTTP webhook from Jotform on submission.  
  - Outputs: Raw JSON containing all submitted form fields and metadata.  
  - Potential Failures: Webhook delivery issues, form ID misconfiguration, API authentication errors.

- **Extract Feedback Data**  
  - Type: Set  
  - Role: Extracts and renames key feedback fields from raw Jotform JSON for clarity and ease of use.  
  - Configuration: Assigns guestName, guestEmail, roomNumber, stayDates, overallRating, feedbackText, serviceArea, submissionId from rawRequest fields.  
  - Expressions: Uses expressions like `={{ $json.rawRequest['q3_guestName'] }}` to map form fields.  
  - Inputs: Output of Jotorm Trigger.  
  - Outputs: Structured JSON with normalized keys.  
  - Edge Cases: Missing or malformed form fields could cause empty or incorrect data.

---

#### 1.2 AI Sentiment Analysis

**Overview:**  
Analyzes the guest feedback text using GPT-4 Langchain Agent to generate a detailed sentiment report including urgency, emotional tone, key issues, and suggested recovery actions.

**Nodes Involved:**  
- Sticky Note2 (comment only)  
- AI Sentiment Analysis  
- Parse AI Response  

**Node Details:**  

- **AI Sentiment Analysis**  
  - Type: Langchain Agent  
  - Role: Sends guest feedback data to a conversational AI agent specialized in hotel service quality analysis.  
  - Configuration:  
    - Prompt includes guest name, room, rating, service area, and feedback text.  
    - Output JSON format specified with fields like sentiment, score, urgencyLevel, keyIssues, emotionalTone, requiresImmediateAction, suggestedRecoveryAction, estimatedImpactOnReputation, categories, summary.  
    - System message sets expert persona for objective and actionable output.  
  - Inputs: Structured feedback data from Extract Feedback Data.  
  - Outputs: AI-generated JSON sentiment analysis.  
  - Potential Failures: API rate limits, malformed prompt data, unexpected AI response format.

- **Parse AI Response**  
  - Type: Set  
  - Role: Prepares AI output JSON for condition checks and routing.  
  - Configuration: Expression data mode to parse and flatten AI JSON response.  
  - Inputs: AI Sentiment Analysis output.  
  - Outputs: JSON object with sentiment analysis data for logical evaluation.

---

#### 1.3 Urgency Routing

**Overview:**  
Checks if the feedback requires immediate escalation and sends alerts to hotel management via Gmail and Slack if urgency is critical or high.

**Nodes Involved:**  
- Sticky Note3 (comment)  
- Check Urgency Level  
- Alert Manager - Urgent (Gmail)  
- Slack Notification  

**Node Details:**  

- **Check Urgency Level**  
  - Type: If  
  - Role: Evaluates conditions to detect critical/high urgency or immediate action requirement.  
  - Conditions:  
    - urgencyLevel equals "critical" OR  
    - urgencyLevel equals "high" OR  
    - requiresImmediateAction equals true  
  - Inputs: Parsed AI sentiment JSON.  
  - Outputs: Branches to alert nodes if true.  
  - Edge Cases: Possible false positives/negatives if AI misclassifies urgency.

- **Alert Manager - Urgent (Gmail)**  
  - Type: Gmail  
  - Role: Sends an HTML-formatted urgent alert email to hotel manager and cc to guest relations team.  
  - Configuration:  
    - Recipient: hotel.manager@yourhotel.com  
    - CC: guestrelations@yourhotel.com  
    - Subject and body include guest info, sentiment details, key issues, and recovery action.  
  - Inputs: Condition true branch from Check Urgency Level, uses data from Extract Feedback Data and Parse AI Response.  
  - Credentials: Gmail OAuth2 configured.  
  - Potential Failures: Email sending errors, authentication issues.

- **Slack Notification**  
  - Type: HTTP Request  
  - Role: Posts JSON payload to Slack webhook URL for real-time communication.  
  - Configuration: POST request to Slack incoming webhook URL with alert details.  
  - Inputs: Same as Gmail alert branch.  
  - Edge Cases: Webhook URL must be valid and active; network issues may cause failure.

---

#### 1.4 Service Recovery

**Overview:**  
Triggers personalized recovery offers and PMS ticket creation for guests with negative sentiment or low overall ratings.

**Nodes Involved:**  
- Sticky Note4 (comment)  
- Check Service Recovery Needed  
- Generate Recovery Offer  
- Parse Recovery Offer  
- Send Recovery Email (Gmail)  
- Create PMS Ticket  

**Node Details:**  

- **Check Service Recovery Needed**  
  - Type: If  
  - Role: Determines if recovery flow should trigger based on:  
    - sentiment equals "negative" OR  
    - overallRating ≤ 3  
  - Inputs: Parsed AI Response and Extract Feedback Data.  
  - Outputs: True branch triggers recovery nodes.  
  - Edge Cases: May miss borderline cases if AI sentiment misclassifies.

- **Generate Recovery Offer**  
  - Type: Langchain Agent  
  - Role: Generates a tailored service recovery offer using GPT-4, considering feedback severity and guest data.  
  - Configuration:  
    - Prompt includes guest name, rating, sentiment score, key issues, service area.  
    - Output JSON with offerType, offerValue, offerDescription, emailSubject, emailBody (HTML), validityPeriod, additionalPerks.  
    - System message frames AI as guest relations expert.  
  - Inputs: Triggered from positive service recovery condition.  
  - Outputs: JSON offer data.  
  - Failures: AI output formatting errors, rate limits.

- **Parse Recovery Offer**  
  - Type: Set  
  - Role: Parses AI-generated recovery offer JSON for use in email and ticket creation.  
  - Inputs: Output of Generate Recovery Offer node.  
  - Outputs: Parsed offer data.

- **Send Recovery Email (Gmail)**  
  - Type: Gmail  
  - Role: Sends personalized recovery email to guest using AI-generated subject and body.  
  - Configuration:  
    - Recipient: guestEmail from Extract Feedback Data  
    - Subject and message body from Parse Recovery Offer  
  - Credentials: Gmail OAuth2.  
  - Failures: Email delivery errors, invalid email addresses.

- **Create PMS Ticket**  
  - Type: HTTP Request  
  - Role: Posts ticket data to Property Management System API for follow-up tracking.  
  - Configuration:  
    - Method: POST  
    - URL: PMS API endpoint (placeholder URL)  
    - Authentication: HTTP header auth with predefined credentials  
    - Payload includes guest details, sentiment analysis, key issues, priority, suggested actions.  
  - Inputs: Triggered alongside Generate Recovery Offer.  
  - Failures: API errors, authentication failures, network issues.

---

#### 1.5 Positive Feedback Handling

**Overview:**  
For positive feedback and high ratings, sends thank-you emails to guests encouraging reviews and offering return discounts, turning satisfied guests into advocates.

**Nodes Involved:**  
- Sticky Note7 (comment)  
- Check Positive Feedback  
- Send Thank You Email (Gmail)  

**Node Details:**  

- **Check Positive Feedback**  
  - Type: If  
  - Role: Checks if sentiment equals "positive" AND overallRating ≥ 4.  
  - Inputs: Parsed AI Response and Extract Feedback Data.  
  - Outputs: True branch triggers thank-you email.  
  - Edge Cases: AI misclassification might miss some positive feedback.

- **Send Thank You Email (Gmail)**  
  - Type: Gmail  
  - Role: Sends personalized thank-you email with embedded HTML content.  
  - Configuration:  
    - Recipient: guestEmail  
    - Subject line includes guest name.  
    - Body encourages online reviews on Google, TripAdvisor, Booking.com, and offers 15% discount with promo code.  
  - Credentials: Gmail OAuth2.  
  - Failures: Email delivery issues, invalid addresses.

---

#### 1.6 Analytics & Logging

**Overview:**  
Logs all guest feedback and AI analysis results into a Google Sheet for performance tracking, trend analysis, and staff training.

**Nodes Involved:**  
- Sticky Note8 (comment)  
- Log to Analytics Sheet  

**Node Details:**  

- **Log to Analytics Sheet**  
  - Type: Google Sheets  
  - Role: Appends or updates a row in the specified Google Sheet with feedback data, AI sentiment, key issues, categories, urgency, and recovery status.  
  - Configuration:  
    - Document ID and Sheet Name specify target Google Sheet.  
    - Maps multiple columns to variables extracted or computed earlier.  
    - Uses USER_ENTERED cell format.  
  - Credentials: Google Sheets OAuth2.  
  - Failures: API limits, authentication failures, permission issues on the spreadsheet.

---

### 3. Summary Table

| Node Name                     | Node Type                           | Functional Role                      | Input Node(s)                  | Output Node(s)                                | Sticky Note                                            |
|-------------------------------|-----------------------------------|------------------------------------|-------------------------------|-----------------------------------------------|--------------------------------------------------------|
| Sticky Note                   | Sticky Note                       | Workflow purpose overview           |                               |                                               | ## 🏨 Hotel Guest Feedback Workflow ...                  |
| Sticky Note1                  | Sticky Note                       | Lists expected Jotform fields       |                               |                                               | ## 📝 Jotform Fields Expected ...                         |
| Jotorm Trigger               | JotForm Trigger                   | Receives form submissions           |                               | Extract Feedback Data                          |                                                        |
| Extract Feedback Data        | Set                              | Extracts and normalizes form fields | Jotorm Trigger                | AI Sentiment Analysis                          |                                                        |
| Sticky Note2                 | Sticky Note                       | AI Sentiment Analysis explanation   |                               |                                               | ## 🤖 AI Sentiment Analysis ...                           |
| AI Sentiment Analysis        | Langchain Agent                   | Generates structured sentiment report| Extract Feedback Data         | Parse AI Response                              |                                                        |
| Parse AI Response            | Set                              | Parses AI JSON response              | AI Sentiment Analysis         | Check Urgency Level, Check Service Recovery Needed, Check Positive Feedback, Log to Analytics Sheet |                                                        |
| Sticky Note3                 | Sticky Note                       | Urgency check explanation            |                               |                                               | ## 🚨 Urgency Check ...                                   |
| Check Urgency Level          | If                               | Routes critical/high urgency issues | Parse AI Response             | Alert Manager - Urgent (Gmail), Slack Notification |                                                        |
| Alert Manager - Urgent (Gmail)| Gmail                           | Sends urgent email to management    | Check Urgency Level           |                                               |                                                        |
| Slack Notification           | HTTP Request                     | Sends Slack alert                   | Check Urgency Level           |                                               |                                                        |
| Sticky Note4                 | Sticky Note                       | Service recovery explanation         |                               |                                               | ## 🔄 Service Recovery Check ...                          |
| Check Service Recovery Needed| If                               | Triggers service recovery flow      | Parse AI Response             | Generate Recovery Offer, Create PMS Ticket     |                                                        |
| Generate Recovery Offer      | Langchain Agent                   | Creates personalized recovery offer | Check Service Recovery Needed | Parse Recovery Offer                           |                                                        |
| Parse Recovery Offer         | Set                              | Parses recovery offer JSON          | Generate Recovery Offer       | Send Recovery Email (Gmail)                    |                                                        |
| Send Recovery Email (Gmail) | Gmail                            | Sends recovery email to guest       | Parse Recovery Offer          |                                               |                                                        |
| Create PMS Ticket            | HTTP Request                     | Creates ticket in PMS system        | Check Service Recovery Needed |                                               |                                                        |
| Sticky Note7                 | Sticky Note                       | Positive feedback handling explanation|                               |                                               | ## ⭐ Positive Feedback Handler ...                       |
| Check Positive Feedback      | If                               | Detects positive feedback            | Parse AI Response             | Send Thank You Email (Gmail)                    |                                                        |
| Send Thank You Email (Gmail) | Gmail                            | Sends thank-you email to guest      | Check Positive Feedback       |                                               |                                                        |
| Sticky Note8                 | Sticky Note                       | Analytics & tracking explanation     |                               |                                               | ## 📊 Analytics & Tracking ...                            |
| Log to Analytics Sheet       | Google Sheets                    | Logs feedback and analysis results  | Parse AI Response, Extract Feedback Data, Check Service Recovery Needed |                                               |                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Jotform Trigger Node**  
   - Node Type: JotForm Trigger  
   - Set webhook for form ID `252862984356471`.  
   - Configure Jotform API credentials.  

2. **Create Set Node "Extract Feedback Data"**  
   - Map raw Jotform submission fields to normalized variables: guestName, guestEmail, roomNumber, stayDates, overallRating (number), feedbackText, serviceArea, submissionId.  
   - Use expressions like `={{ $json.rawRequest['q3_guestName'] }}`.

3. **Create Langchain Agent Node "AI Sentiment Analysis"**  
   - Use GPT-4 model (e.g., `gpt-4.1-mini`).  
   - Prompt includes guest details and feedback.  
   - Define expected JSON output schema for sentiment analysis.  
   - Add system message describing expert hotel service analyst role.  
   - Connect input from "Extract Feedback Data".  

4. **Create Set Node "Parse AI Response"**  
   - Set mode to expression data to parse AI JSON output for further logic.  
   - Connect input from "AI Sentiment Analysis".  

5. **Create If Node "Check Urgency Level"**  
   - Define OR conditions: urgencyLevel = "critical" OR "high" OR requiresImmediateAction = true.  
   - Input from "Parse AI Response".  

6. **Create Gmail Node "Alert Manager - Urgent (Gmail)"**  
   - Configure Gmail OAuth2 credentials.  
   - Set recipient to hotel.manager@yourhotel.com, CC guestrelations@yourhotel.com.  
   - Compose HTML email incorporating variables from Extract Feedback Data and Parse AI Response.  
   - Trigger from true branch of "Check Urgency Level".  

7. **Create HTTP Request Node "Slack Notification"**  
   - Set POST method to Slack webhook URL.  
   - Send JSON payload reflecting critical feedback alert.  
   - Trigger also from true branch of "Check Urgency Level".  

8. **Create If Node "Check Service Recovery Needed"**  
   - OR conditions: sentiment = "negative" OR overallRating ≤ 3.  
   - Input from "Parse AI Response".  

9. **Create Langchain Agent Node "Generate Recovery Offer"**  
   - Use GPT-4 model.  
   - Prompt includes guest data, sentiment score, and key issues.  
   - Output JSON with recovery offer details.  
   - System message defines hotel guest relations expert persona.  
   - Trigger from true branch of "Check Service Recovery Needed".  

10. **Create Set Node "Parse Recovery Offer"**  
    - Parse AI-generated recovery offer JSON.  
    - Connected from "Generate Recovery Offer".  

11. **Create Gmail Node "Send Recovery Email (Gmail)"**  
    - Gmail OAuth2 credentials.  
    - Recipient: guestEmail.  
    - Use parsed emailSubject and emailBody from "Parse Recovery Offer".  
    - Trigger from "Parse Recovery Offer".  

12. **Create HTTP Request Node "Create PMS Ticket"**  
    - Configure POST to PMS API URL with HTTP header auth credentials.  
    - Payload includes guest info, sentiment, key issues, priority, suggested actions.  
    - Trigger simultaneously with "Generate Recovery Offer" true branch.  

13. **Create If Node "Check Positive Feedback"**  
    - AND conditions: sentiment = "positive" AND overallRating ≥ 4.  
    - Input from "Parse AI Response".  

14. **Create Gmail Node "Send Thank You Email (Gmail)"**  
    - Gmail OAuth2 credentials.  
    - Recipient: guestEmail.  
    - Compose customized thank-you email with HTML content, review links, and discount code.  
    - Trigger from true branch of "Check Positive Feedback".  

15. **Create Google Sheets Node "Log to Analytics Sheet"**  
    - Configure Google Sheets OAuth2 credentials.  
    - Target sheet: "Guest Feedback Analytics" in specified spreadsheet ID.  
    - Map columns for guest info, sentiment, categories, urgency, recovery status, timestamp, etc.  
    - Append new row for each submission.  
    - Trigger from "Parse AI Response" main output.  

16. **Connect all nodes according to logical flow:**  
    - Jotform Trigger → Extract Feedback Data → AI Sentiment Analysis → Parse AI Response → branching to Check Urgency Level, Check Service Recovery Needed, Check Positive Feedback, Log to Analytics Sheet.  
    - Urgency and recovery branches connect to respective email/slack and PMS ticket nodes.  
    - Positive feedback branch connects to thank-you email node.  

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                         |
|---------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| Jotform free form creation link for guest feedback capture.                                                    | https://www.jotform.com/?partner=mediajade                                                            |
| Workflow achieves 60% reduction in negative reviews through real-time automated service recovery.              | Workflow Purpose Sticky Note                                                                            |
| Google Sheets used for analytics and tracking, enabling trend analysis and staff training insights.            | Analytics Sticky Note                                                                                   |
| Slack webhook URL must be replaced with actual workspace webhook for notifications to function correctly.      | Slack Notification Node Configuration                                                                  |
| PMS ticket creation requires valid API endpoint and authentication credentials for integration.                 | Create PMS Ticket Node                                                                                   |
| Gmail OAuth2 credentials must be set up with appropriate scopes for sending emails.                             | Gmail nodes across the workflow                                                                         |
| OpenAI GPT-4 model `gpt-4.1-mini` used for all AI-driven nodes to balance capability and cost.                 | AI Sentiment Analysis and Generate Recovery Offer nodes                                                 |

---

Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.