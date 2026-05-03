Automated Customer Feedback Analysis & Response with Jotform, GPT-4.1 & Google Sheets

https://n8nworkflows.xyz/workflows/automated-customer-feedback-analysis---response-with-jotform--gpt-4-1---google-sheets-9623


# Automated Customer Feedback Analysis & Response with Jotform, GPT-4.1 & Google Sheets

### 1. Workflow Overview

This workflow automates the analysis and response process for customer feedback submitted via a Jotform feedback form. It leverages AI (GPT-4.1) to classify sentiment, identify root causes, and generate personalized recovery or appreciation messages. Based on sentiment and rating, it routes feedback into two main paths: Recovery Path for negative or low-rating feedback, and Appreciation Path for positive or neutral feedback. Both paths log detailed feedback data into Google Sheets and notify appropriate teams or customers via email and Slack.

Logical blocks:

- **1.1 Input Reception**  
  Captures new feedback submissions from Jotform and normalizes incoming data fields.

- **1.2 AI Processing**  
  Analyzes feedback sentiment and root cause using GPT-4.1, then routes feedback based on negative sentiment or low ratings.

- **1.3 Negative Feedback Handling (Recovery Path)**  
  Generates personalized recovery messages, logs data into Google Sheets, sends recovery emails, notifies CX team via Slack, and updates the sheet about message dispatch.

- **1.4 Positive/Neutral Feedback Handling (Appreciation Path)**  
  Prepares and sends thank-you emails with a call-to-action for public reviews, logs data into Google Sheets.

- **1.5 Supportive Documentation and Notes**  
  Sticky notes provide configuration guides, API credential instructions, and workflow summary.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block triggers on new Jotform submissions and normalizes the raw input data into structured variables for easier downstream processing.

- **Nodes Involved:**  
  - Jotform — New Feedback Submission  
  - Extract Key Fields (Normalize Input)  

- **Node Details:**  

  - **Jotform — New Feedback Submission**  
    - Type: JotForm Trigger Node  
    - Role: Listens for new submissions on a specific Jotform (form ID: 251411842823048)  
    - Configuration: Full form answers captured (onlyAnswers: false)  
    - Credentials: Uses JotForm API credentials (Full Access recommended)  
    - Inputs: Webhook trigger (new form submission)  
    - Outputs: Raw submission JSON with all fields  
    - Edge Cases: API connectivity issues, webhook failures, missing fields if form changes  
    - Sticky Note: Instructions on obtaining Jotform API credentials and form setup  

  - **Extract Key Fields (Normalize Input)**  
    - Type: Set Node  
    - Role: Maps raw Jotform fields to normalized variables with consistent naming for downstream use  
    - Configuration: Extracts submissionID, fullName, email, whatsappNumber, orderId, ratings, experienceFeedback, recommendUs from raw JSON  
    - Expressions: Uses expressions like `={{ $json.rawRequest["Full Name"] }}`  
    - Inputs: From Jotform Trigger  
    - Outputs: Cleaned, consistently named JSON object  
    - Edge Cases: Missing or malformed fields, expression evaluation errors  
    - Sticky Note: Describes the normalization purpose and key fields extracted  

#### 2.2 AI Processing

- **Overview:**  
  Analyzes customer feedback using GPT-4.1 to determine sentiment, root cause, and recovery directives, then routes the feedback to either recovery or appreciation paths.

- **Nodes Involved:**  
  - AI Analysis — Sentiment & Root Cause  
  - Check if Feedback is Negative or Rating ≤ 3  

- **Node Details:**  

  - **AI Analysis — Sentiment & Root Cause**  
    - Type: LangChain OpenAI Node  
    - Role: Sends feedback details to GPT-4.1 model for structured sentiment and root cause analysis  
    - Configuration:  
      - Model: GPT-4.1-mini  
      - Temperature: 0.7 (balanced creativity)  
      - Prompt: Detailed instructions to return JSON with Sentiment, RootCause, RecoveryDirection, RecoveryMessage only  
      - Inputs: rating, experience feedback, recommendation answer from normalized data  
    - Outputs: JSON object with keys Sentiment, RootCause, RecoveryDirection, RecoveryMessage  
    - Edge Cases: API rate limits, malformed responses, timeouts, model errors, incomplete JSON output  
    - Version: Requires LangChain-compatible OpenAI node version 1.8+  

  - **Check if Feedback is Negative or Rating ≤ 3**  
    - Type: If Node  
    - Role: Decision logic to route feedback  
    - Configuration:  
      - Condition: Sentiment equals "Negative" OR rating less than or equal to 3  
      - Case insensitive, loose type validation  
    - Inputs: Output from AI Analysis node  
    - Outputs:  
      - True: Negative feedback path (recovery)  
      - False: Positive/neutral feedback path (appreciation)  
    - Edge Cases: Missing sentiment or rating fields causing false routing, expression evaluation errors  
    - Sticky Note: Explains routing logic clearly  

#### 2.3 Negative Feedback Handling (Recovery Path)

- **Overview:**  
  For negative or low-rating feedback, generates personalized recovery messages, logs feedback, sends recovery emails, notifies CX team via Slack, and marks the message as sent in Google Sheets.

- **Nodes Involved:**  
  - AI Generator — Personalized Recovery Message  
  - Prepare Data for Sheet & Notifications (Negative Path)  
  - Log Feedback in Google Sheet (Negative Path)  
  - Send Recovery Email (Negative Path)  
  - Notify CX Team on Slack  
  - Mark Recovery Message Sent  

- **Node Details:**  

  - **AI Generator — Personalized Recovery Message**  
    - Type: LangChain OpenAI Node  
    - Role: Generates empathetic, personalized recovery email drafts based on AI analysis and customer data  
    - Configuration:  
      - Model: GPT-4.1-mini  
      - Temperature: 0.7  
      - Prompt: Compose JSON with Email Body, Subject, Tone using customer name, brand, sentiment, root cause, feedback, etc.  
      - Output: JSON only, no markdown or extra text  
    - Inputs: Customer data from normalized input and AI analysis output  
    - Outputs: Recovery message JSON  
    - Edge Cases: API errors, incomplete JSON, long or off-brand messages  
    - Version: LangChain OpenAI 1.8+  

  - **Prepare Data for Sheet & Notifications (Negative Path)**  
    - Type: Set Node  
    - Role: Consolidates all relevant fields including AI outputs to prepare for logging and notifications  
    - Configuration: Assigns submission ID, contact info, feedback, AI sentiment/analysis, recovery message details, timestamp  
    - Inputs: Normalized data, AI analysis, AI generated recovery message  
    - Outputs: Structured data object ready for Google Sheets and Slack  
    - Edge Cases: Missing fields, timestamp conversion issues  

  - **Log Feedback in Google Sheet (Negative Path)**  
    - Type: Google Sheets Node  
    - Role: Appends feedback and recovery data to a configured Google Sheet for record-keeping  
    - Configuration:  
      - Operation: Append row  
      - Sheet: gid=0 (configurable)  
      - Document ID: Your Google Sheet document ID required  
      - Authentication: Service Account  
      - Auto mapping of input data to sheet columns  
    - Inputs: Prepared data from previous node  
    - Outputs: Confirmation of append operation  
    - Edge Cases: Authentication failure, rate limits, incorrect sheet ID, schema mismatch  

  - **Send Recovery Email (Negative Path)**  
    - Type: Email Send Node  
    - Role: Sends the AI-generated personalized recovery email to the customer  
    - Configuration:  
      - From: Configured SMTP email (example: aayushmans0411@gmail.com)  
      - To: Customer email from normalized data  
      - Subject and HTML body from AI Generator output  
    - Inputs: Email content from AI Generator node  
    - Outputs: Email send confirmation  
    - Edge Cases: SMTP errors, invalid email addresses, email delivery failures  

  - **Notify CX Team on Slack**  
    - Type: Slack Node  
    - Role: Sends alert message to CX team Slack channel about negative feedback and recovery action  
    - Configuration:  
      - Channel: #customer_feedback  
      - Message includes customer name, rating, root cause, order ID, and confirmation of recovery message sent  
      - OAuth or token-based Slack credentials  
    - Inputs: Data from prepared data node  
    - Outputs: Slack message confirmation  
    - Edge Cases: Slack API errors, channel access issues  

  - **Mark Recovery Message Sent**  
    - Type: Google Sheets Node  
    - Role: Updates the original feedback row in Google Sheets to mark that recovery message was sent  
    - Configuration:  
      - Operation: Update row based on Submission Id  
      - Sets “Recovery Message Sent” column to “Yes”  
      - Authentication: Service Account  
    - Inputs: Feedback submission ID from prepared data  
    - Outputs: Update confirmation  
    - Edge Cases: Row not found, authentication failure, concurrency issues  

  - **Sticky Note:**  
    - Detailed AI recovery message composition guide and negative path flow overview  

#### 2.4 Positive/Neutral Feedback Handling (Appreciation Path)

- **Overview:**  
  For positive or neutral feedback, prepares and sends a personalized thank-you email, includes a call-to-action for public reviews, and logs feedback data in Google Sheets.

- **Nodes Involved:**  
  - Prepare Data for Sheet & Notifications (Positive Path)  
  - Log Feedback in Google Sheet (Positive Path)  
  - Send Appreciation Email (Positive Path)  

- **Node Details:**  

  - **Prepare Data for Sheet & Notifications (Positive Path)**  
    - Type: Set Node  
    - Role: Prepares structured data combining normalized input and AI analysis for logging and email  
    - Configuration: Maps key fields including submission ID, contact info, feedback, sentiment, root cause, and timestamp  
    - Inputs: Normalized data and AI analysis output  
    - Outputs: JSON object for logging and emailing  
    - Edge Cases: Missing fields, timestamp format issues  

  - **Log Feedback in Google Sheet (Positive Path)**  
    - Type: Google Sheets Node  
    - Role: Appends positive feedback data to Google Sheets for record  
    - Configuration:  
      - Operation: Append row  
      - Sheet: gid=0  
      - Document ID: Configured Google Sheets document ID  
      - Authentication: Service Account  
    - Inputs: Prepared data node output  
    - Outputs: Append confirmation  
    - Edge Cases: Authentication or API errors, schema mismatch  

  - **Send Appreciation Email (Positive Path)**  
    - Type: Email Send Node  
    - Role: Sends a branded thank-you email with a request for customers to leave public reviews  
    - Configuration:  
      - From: Configured SMTP email  
      - To: Customer's email  
      - Subject: Fixed subject encouraging sharing experience  
      - HTML Body: Static well-designed email template with dynamic insertion of customer name  
    - Inputs: Prepared data for customer email  
    - Outputs: Email send confirmation  
    - Edge Cases: SMTP delivery errors, invalid email addresses  

  - **Sticky Note:**  
    - Appreciation path flow and objectives explained  

#### 2.5 Supportive Documentation and Notes

- **Overview:**  
  Contains sticky notes with workflow summary, API credential setup, Jotform configuration instructions, and block explanations.

- **Nodes Involved:**  
  - Sticky Note (Workflow Summary)  
  - Sticky Note1 (Data Normalization)  
  - Sticky Note2 (AI Analysis Explanation)  
  - Sticky Note3 (Routing Logic)  
  - Sticky Note4 (Recovery Message AI & Negative Path Details)  
  - Sticky Note5 (Appreciation Path Details)  
  - Sticky Note6 (Jotform API Credentials Guide)  
  - Sticky Note7 (Jotform Form Setup Instructions)  

- **Role:**  
  Provides essential contextual information and configuration instructions for users building or maintaining the workflow.

---

### 3. Summary Table

| Node Name                                  | Node Type                 | Functional Role                                         | Input Node(s)                         | Output Node(s)                                  | Sticky Note                                                                                                                          |
|--------------------------------------------|---------------------------|--------------------------------------------------------|-------------------------------------|------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| Jotform — New Feedback Submission           | JotForm Trigger           | Trigger on new feedback submission                      | Webhook trigger                    | Extract Key Fields (Normalize Input)            | How to get Jotform API Credentials, Jotform Setup Guide                                                                              |
| Extract Key Fields (Normalize Input)        | Set Node                  | Normalize raw feedback data                             | Jotform — New Feedback Submission  | AI Analysis — Sentiment & Root Cause             | Data normalization explanation                                                                                                       |
| AI Analysis — Sentiment & Root Cause        | LangChain OpenAI          | Analyze sentiment, root cause, recovery directives     | Extract Key Fields (Normalize Input) | Check if Feedback is Negative or Rating ≤ 3      | AI sentiment and root cause detection explanation                                                                                     |
| Check if Feedback is Negative or Rating ≤ 3 | If Node                   | Routes feedback to negative or positive path            | AI Analysis — Sentiment & Root Cause | AI Generator — Personalized Recovery Message (True), Prepare Data for Sheet & Notifications (Positive Path) (False) | Routing logic explanation                                                                                                            |
| AI Generator — Personalized Recovery Message | LangChain OpenAI          | Generate personalized recovery message                  | Check if Feedback is Negative or Rating ≤ 3 (True) | Prepare Data for Sheet & Notifications (Negative Path) | AI recovery message composition guide                                                                                               |
| Prepare Data for Sheet & Notifications (Negative Path) | Set Node                  | Prepare data for logging, email, and notifications     | AI Generator — Personalized Recovery Message | Log Feedback in Google Sheet (Negative Path), Send Recovery Email (Negative Path), Notify CX Team on Slack |                                                                                                                                        |
| Log Feedback in Google Sheet (Negative Path) | Google Sheets             | Append negative feedback data to Google Sheet           | Prepare Data for Sheet & Notifications (Negative Path) | Send Recovery Email (Negative Path), Notify CX Team on Slack |                                                                                                                                        |
| Send Recovery Email (Negative Path)          | Email Send                | Send recovery email to customer                          | Prepare Data for Sheet & Notifications (Negative Path) | Mark Recovery Message Sent                       |                                                                                                                                        |
| Notify CX Team on Slack                       | Slack                     | Notify CX team about negative feedback                   | Prepare Data for Sheet & Notifications (Negative Path) |                                                |                                                                                                                                        |
| Mark Recovery Message Sent                    | Google Sheets             | Update sheet to mark recovery message sent              | Send Recovery Email (Negative Path) |                                                |                                                                                                                                        |
| Prepare Data for Sheet & Notifications (Positive Path) | Set Node                  | Prepare data for logging and appreciation email         | Check if Feedback is Negative or Rating ≤ 3 (False) | Log Feedback in Google Sheet (Positive Path), Send Appreciation Email (Positive Path) |                                                                                                                                        |
| Log Feedback in Google Sheet (Positive Path) | Google Sheets             | Append positive feedback data to Google Sheet            | Prepare Data for Sheet & Notifications (Positive Path) | Send Appreciation Email (Positive Path)          |                                                                                                                                        |
| Send Appreciation Email (Positive Path)       | Email Send                | Send thank-you email with review CTA                      | Prepare Data for Sheet & Notifications (Positive Path) |                                                |                                                                                                                                        |
| Sticky Note                                   | Sticky Note               | Workflow summary and overview                            |                                     |                                                | Provides a concise workflow summary and link to sample Google Sheet template                                                         |
| Sticky Note1                                  | Sticky Note               | Data normalization explanation                           |                                     |                                                | Explains purpose of the normalization node                                                                                           |
| Sticky Note2                                  | Sticky Note               | AI sentiment and root cause explanation                  |                                     |                                                | Explains AI analysis node’s output and prompt                                                                                        |
| Sticky Note3                                  | Sticky Note               | Routing logic description                               |                                     |                                                | Details decision node logic for routing feedback                                                                                     |
| Sticky Note4                                  | Sticky Note               | AI recovery message and negative path flow              |                                     |                                                | Describes AI recovery message composition and negative feedback sub-workflow                                                        |
| Sticky Note5                                  | Sticky Note               | Appreciation path flow explanation                       |                                     |                                                | Describes positive/neutral feedback handling and objectives                                                                         |
| Sticky Note6                                  | Sticky Note               | Jotform API credentials setup guide                      |                                     |                                                | Step-by-step instructions to obtain Jotform API key                                                                                 |
| Sticky Note7                                  | Sticky Note               | Jotform form configuration instructions                 |                                     |                                                | Exact form field labels and types required for proper workflow operation                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Jotform Feedback Form**  
   - Fields: Full Name (Short Text), Email (Email), Whatsapp Number (Short Text), Order Id (Short Text), Ratings (Rating 1–5), Please describe your experience in detail (Long Text), Would you recommend us to others? (Single Choice Yes/No)  
   - Keep exact field labels as above.

2. **Set up Jotform API Credentials in n8n**  
   - Generate API key from Jotform account (full access recommended)  
   - Create new JotForm API credential in n8n with the key.

3. **Add Jotform Trigger Node**  
   - Type: JotForm Trigger  
   - Select your feedback form by ID  
   - Use JotForm API credentials  
   - This node listens for new submissions.

4. **Add Set Node to Normalize Input**  
   - Type: Set  
   - Create fields: submissionId, fullName, email, whatsappNumber, orderId, ratings, experienceFeedback, recommendUs  
   - Assign values using expressions from raw Jotform JSON fields (e.g. `={{ $json.rawRequest["Full Name"] }}`).

5. **Add LangChain OpenAI Node for Sentiment & Root Cause Analysis**  
   - Model: GPT-4.1-mini  
   - Temperature: 0.7  
   - Prompt: Provide customer feedback details and instruct AI to return JSON with Sentiment, RootCause, RecoveryDirection, RecoveryMessage only  
   - Use normalized fields as input context.

6. **Add If Node for Routing**  
   - Condition: Sentiment equals "Negative" OR ratings ≤ 3  
   - True branch: negative feedback path  
   - False branch: positive/neutral feedback path.

7. **Negative Feedback Path**

   1. **Add LangChain OpenAI Node to Generate Recovery Message**  
      - Model: GPT-4.1-mini  
      - Temperature: 0.7  
      - Prompt: Use customer data and AI analysis to generate personalized JSON recovery message including Email Body, Subject, Tone.

   2. **Add Set Node to Prepare Data for Logging and Notifications**  
      - Combine normalized input, AI analysis, recovery message, and timestamp into one object.

   3. **Add Google Sheets Node to Log Negative Feedback**  
      - Operation: Append row  
      - Configure Google Sheets document ID and sheet name (gid=0)  
      - Use Service Account authentication  
      - Map input fields to sheet columns.

   4. **Add Email Send Node to Send Recovery Email**  
      - From: Your verified SMTP email  
      - To: Customer email from normalized data  
      - Subject and HTML from AI-generated recovery message.

   5. **Add Slack Node to Notify CX Team**  
      - Channel: #customer_feedback  
      - Message includes customer name, rating, root cause, order ID, and note that recovery message was sent.

   6. **Add Google Sheets Node to Mark Recovery Message Sent**  
      - Operation: Update row based on Submission Id  
      - Set “Recovery Message Sent” column to “Yes”.

8. **Positive/Neutral Feedback Path**

   1. **Add Set Node to Prepare Data for Logging and Email**  
      - Map normalized input, AI analysis, and timestamp.

   2. **Add Google Sheets Node to Log Positive Feedback**  
      - Operation: Append row  
      - Configure same Google Sheet and authentication.

   3. **Add Email Send Node to Send Appreciation Email**  
      - From: Your SMTP email  
      - To: Customer email  
      - Subject: Fixed thank-you subject  
      - HTML: Static branded email template with dynamic customer name insertion.

9. **Finalize Connections**  
   - Connect nodes in the order described ensuring correct data propagation and branching.

10. **Test Workflow**  
    - Submit test entries in Jotform and verify data flows, AI outputs, emails, Slack notifications, and Google Sheets logging.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| Sample Google Sheets template for feedback logging: [Copy Template](https://docs.google.com/spreadsheets/u/2/d/1YYmyQNTGSdBQcoHuUI1tnd081Nq-5FVcN8KWGLf0iK8/copy) | Workflow summary sticky note                                                                                      |
| Jotform API credential generation steps with full access recommended                                                                       | See Sticky Note6 content                                                                                          |
| Exact Jotform form field requirements and labels (case-sensitive)                                                                           | See Sticky Note7 content                                                                                          |
| AI prompt design emphasizes JSON-only output for seamless parsing                                                                          | Embedded in AI Analysis and AI Generator nodes                                                                   |
| Slack channel should be pre-created and accessible by OAuth token or Slack app credentials                                                 | Slack node configuration note                                                                                     |
| SMTP account must be verified and allowed to send emails from configured “From” address                                                     | Email Send nodes configuration                                                                                     |
| Timestamp stored as local milliseconds since epoch for logging and auditing                                                                 | Set nodes use expression `$now.toLocal().toMillis()`                                                              |
| Brand name and links in email templates should be customized to your organization                                                          | Appreciation email HTML content includes placeholders                                                             |

---

This documentation enables comprehensive understanding, stepwise reproduction, and maintenance of the "Automated Customer Feedback Analysis & Response with Jotform, GPT-4.1 & Google Sheets" workflow in n8n.