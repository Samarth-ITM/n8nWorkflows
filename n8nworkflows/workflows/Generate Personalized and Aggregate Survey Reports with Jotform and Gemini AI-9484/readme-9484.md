Generate Personalized and Aggregate Survey Reports with Jotform and Gemini AI

https://n8nworkflows.xyz/workflows/generate-personalized-and-aggregate-survey-reports-with-jotform-and-gemini-ai-9484


# Generate Personalized and Aggregate Survey Reports with Jotform and Gemini AI

### 1. Workflow Overview

This n8n workflow automates the generation and distribution of personalized and aggregate survey reports based on data collected via Jotform. It targets organizations or researchers conducting surveys who want immediate personalized feedback to respondents and a weekly statistical summary report sent to administrators.

The workflow is logically divided into two main functional blocks:

- **1.1 Individual Survey Response Processing:** Triggered instantly when a respondent submits a survey, it generates and emails a personalized report highlighting key insights and recommendations specific to that individual.

- **1.2 Weekly Aggregate Reporting:** Runs on a weekly schedule to fetch all survey submissions, normalize and extract key question-answer pairs, aggregate the data, perform statistical analysis, and then generate and send a comprehensive HTML report to the admin.

---

### 2. Block-by-Block Analysis

#### 2.1 Individual Survey Response Processing

**Overview:**  
This block handles real-time processing of individual survey submissions. Upon receiving a new submission via Jotform webhook, it aggregates the submission data, uses Google Gemini AI to generate a personalized HTML report with insights and recommendations, and then emails that report to the respondent.

**Nodes Involved:**  
- Form Submission Trigger  
- Collect Individual Response  
- Gemini LLM (Personal)  
- Generate Personal Report  
- Parse Personal Report JSON  
- Response Ready - Pause  
- Send Personal Report  

**Node Details:**

- **Form Submission Trigger**  
  - *Type:* Jotform Trigger  
  - *Role:* Initiates workflow on new survey submission  
  - *Config:* Listens on webhook for specific Jotform form ID  
  - *Input:* Webhook HTTP POST from Jotform  
  - *Output:* JSON survey data from submission  
  - *Failure Modes:* Webhook misconfiguration, form ID mismatch, network issues  

- **Collect Individual Response**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all item data from the single submission (effectively passes data forward)  
  - *Config:* Aggregates all input data items  
  - *Input:* Output from Form Submission Trigger  
  - *Output:* Aggregated JSON data for AI processing  
  - *Failure Modes:* Empty or malformed submission payload  

- **Gemini LLM (Personal)**  
  - *Type:* Google Gemini Language Model Node  
  - *Role:* Interface node for AI model to process the prompt for personal report generation  
  - *Config:* Uses default options, no special parameters here  
  - *Input:* Text prompt defining instructions and including respondent data  
  - *Output:* Raw AI-generated text response (JSON string expected)  
  - *Failure Modes:* API key/credential issues, rate limits, network failure  

- **Generate Personal Report**  
  - *Type:* Langchain Agent Node  
  - *Role:* Constructs and sends the detailed prompt to Gemini LLM for personalized report generation  
  - *Config:*  
    - Prompt includes: today's date, respondent data JSON, instructions to extract demographics, analyze responses, create HTML report with styling  
    - System message guides AI to concise, bullet-pointed output with professional styling requirements  
    - Output parser enabled for JSON only  
  - *Input:* Aggregated response data  
  - *Output:* AI-generated JSON with "email_subject" and "html_report"  
  - *Failure Modes:* Expression evaluation errors, AI returning invalid JSON, malformed input data  

- **Parse Personal Report JSON**  
  - *Type:* Langchain Output Parser (Structured)  
  - *Role:* Parses AI output strictly as JSON to extract specific fields  
  - *Config:* Expects a JSON schema with "email_subject" (string) and "html_report" (string)  
  - *Input:* Raw AI output text from Generate Personal Report  
  - *Output:* Parsed JSON object with extracted fields ready for email  
  - *Failure Modes:* Parsing errors if AI output is malformed  

- **Response Ready - Pause**  
  - *Type:* Wait  
  - *Role:* Acts as a controlled pause or buffer before sending the email  
  - *Config:* No delay configured (default), acts as a logical step  
  - *Input:* Parsed report JSON  
  - *Output:* Passes data forward  
  - *Failure Modes:* None significant  

- **Send Personal Report**  
  - *Type:* Gmail Node  
  - *Role:* Sends the personalized HTML report to the respondent's email address  
  - *Config:*  
    - Recipient email extracted dynamically from submission data field "Please indicate your email so that we can reach you about the results."  
    - Email subject and body from parsed AI output fields  
    - Attribution disabled for clean emails  
  - *Input:* Parsed JSON with subject and HTML report  
  - *Output:* Email sent confirmation  
  - *Failure Modes:* Gmail OAuth2 credential issues, invalid recipient email, SMTP failures  

---

#### 2.2 Weekly Aggregate Reporting

**Overview:**  
This block runs weekly on a schedule to fetch all Jotform survey submissions via API, normalize and extract question-answer pairs from each submission, aggregate the data, invoke Gemini AI to generate a comprehensive statistical report, and finally email this report to the admin.

**Nodes Involved:**  
- Weekly Report Scheduler  
- Fetch Survey Submissions  
- Unpack Response Objects  
- Normalize Response Fields  
- Extract Q&A Pairs  
- Merge All Responses  
- Gemini LLM  
- Analyze & Report  
- Structured Output Parser  
- Send Report to Admin  

**Node Details:**

- **Weekly Report Scheduler**  
  - *Type:* Schedule Trigger  
  - *Role:* Triggers the aggregate reporting process every week at 11 AM  
  - *Config:* Interval set to weekly, trigger time 11:00  
  - *Input:* Schedule event  
  - *Output:* Trigger signal  
  - *Failure Modes:* Server downtime or workflow inactive at trigger time  

- **Fetch Survey Submissions**  
  - *Type:* HTTP Request  
  - *Role:* Retrieves all survey submissions from Jotform API using API key  
  - *Config:*  
    - URL targets submissions endpoint for configured form ID  
    - Query parameter includes Jotform API key  
  - *Input:* Trigger signal from scheduler  
  - *Output:* JSON array of all submissions  
  - *Failure Modes:* API key invalid or expired, HTTP errors, rate limiting, network issues  

- **Unpack Response Objects**  
  - *Type:* Split Out  
  - *Role:* Splits the array of submissions into individual items for processing  
  - *Config:* Splits on field "content", includes "content.id" and "content.answers" fields  
  - *Input:* JSON array of submissions  
  - *Output:* Individual submission objects  
  - *Failure Modes:* Unexpected JSON structure from Jotform API  

- **Normalize Response Fields**  
  - *Type:* Set  
  - *Role:* Normalizes fields to a consistent format for downstream code processing  
  - *Config:* Assigns "content.id" and "content.answers" as strings/objects accordingly  
  - *Input:* Individual submission items  
  - *Output:* Normalized JSON objects  
  - *Failure Modes:* Missing fields in submissions  

- **Extract Q&A Pairs**  
  - *Type:* Code (JavaScript)  
  - *Role:* Extracts question text and answers pairs, filtering out non-question controls  
  - *Config:*  
    - Iterates all input items  
    - For each submission, extracts id and answers where question text and answers exist and type is not header or button  
    - Returns array of objects with id and survey_responses (question-answer pairs)  
  - *Input:* Normalized submission data  
  - *Output:* Array of extracted Q&A pairs per submission  
  - *Failure Modes:* JS runtime errors, unexpected data format  

- **Merge All Responses**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all extracted Q&A pairs into a single array for analysis  
  - *Config:* Aggregate all item data  
  - *Input:* Extracted Q&A pairs  
  - *Output:* Aggregated data array  
  - *Failure Modes:* Empty input, aggregation errors  

- **Gemini LLM**  
  - *Type:* Google Gemini Language Model Node  
  - *Role:* AI interface for the aggregate report generation prompt  
  - *Config:* Default options  
  - *Input:* Aggregated Q&A data embedded in prompt text by next node  
  - *Output:* Raw AI-generated JSON string  
  - *Failure Modes:* API issues, rate limits, invalid input  

- **Analyze & Report**  
  - *Type:* Langchain Agent Node  
  - *Role:* Sends prompt to Gemini AI to generate an aggregate HTML report with statistics and insights  
  - *Config:*  
    - Prompt instructs to analyze all survey data  
    - Requires a JSON output with "email_subject" and "html_report" fields  
    - System message emphasizes professional, statistic-rich, styled HTML output  
    - Output parser enabled  
  - *Input:* Aggregated survey data  
  - *Output:* Parsed JSON with report subject and full HTML report  
  - *Failure Modes:* AI output errors, prompt misconfigurations  

- **Structured Output Parser**  
  - *Type:* Langchain Output Parser (Structured)  
  - *Role:* Parses AI output strictly as JSON to extract subject and HTML report  
  - *Config:* Expects JSON schema with "email_subject" and "html_report" strings  
  - *Input:* Raw AI output text  
  - *Output:* Parsed JSON for email sending  
  - *Failure Modes:* Parsing failures due to malformed AI output  

- **Send Report to Admin**  
  - *Type:* Gmail Node  
  - *Role:* Sends the weekly aggregated HTML report to the configured admin email address  
  - *Config:*  
    - Fixed recipient email (YOUR_ADMIN_EMAIL@example.com)  
    - Email subject and body from parsed AI output  
    - Attribution disabled for professionalism  
  - *Input:* Parsed JSON report  
  - *Output:* Email sent confirmation  
  - *Failure Modes:* Credential issues, invalid admin email, SMTP failures  

---

### 3. Summary Table

| Node Name                 | Node Type                               | Functional Role                              | Input Node(s)                       | Output Node(s)                      | Sticky Note                                                                                              |
|---------------------------|---------------------------------------|----------------------------------------------|-----------------------------------|-----------------------------------|---------------------------------------------------------------------------------------------------------|
| Form Submission Trigger    | Jotform Trigger                       | Initiates on new survey submission           | —                                 | Collect Individual Response         | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Collect Individual Response| Aggregate                            | Aggregates single submission data             | Form Submission Trigger            | Generate Personal Report            | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Gemini LLM (Personal)      | Google Gemini LLM                    | AI interface for personal report generation   | Generate Personal Report           | Generate Personal Report            | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Generate Personal Report   | Langchain Agent                     | Creates personalized report prompt and output | Collect Individual Response        | Response Ready - Pause              | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Parse Personal Report JSON | Langchain Output Parser (Structured)| Parses AI output JSON for personal report     | Generate Personal Report           | Response Ready - Pause              | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Response Ready - Pause     | Wait                                | Buffer step before sending email               | Parse Personal Report JSON         | Send Personal Report                | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Send Personal Report       | Gmail                               | Sends personalized report email to respondent | Response Ready - Pause             | —                                 | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Weekly Report Scheduler    | Schedule Trigger                    | Weekly trigger to start aggregate reporting   | —                                 | Fetch Survey Submissions            | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Fetch Survey Submissions   | HTTP Request                       | Retrieves all survey submissions from Jotform | Weekly Report Scheduler            | Unpack Response Objects             | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Unpack Response Objects    | Split Out                         | Splits array of submissions into individual items | Fetch Survey Submissions         | Normalize Response Fields           | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Normalize Response Fields  | Set                               | Normalizes submission fields for processing   | Unpack Response Objects            | Extract Q&A Pairs                  | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Extract Q&A Pairs          | Code                              | Extracts question-answer pairs from submissions | Normalize Response Fields          | Merge All Responses                | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Merge All Responses        | Aggregate                         | Aggregates all extracted responses            | Extract Q&A Pairs                  | Analyze & Report                   | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Gemini LLM                 | Google Gemini LLM                  | AI interface for aggregate report generation  | Analyze & Report (input)           | Analyze & Report                   | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Analyze & Report           | Langchain Agent                   | Generates aggregate statistical report         | Merge All Responses                | Send Report to Admin               | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Structured Output Parser   | Langchain Output Parser (Structured)| Parses AI output JSON for aggregate report     | Analyze & Report                  | Send Report to Admin               | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Send Report to Admin       | Gmail                             | Sends aggregate report email to admin          | Structured Output Parser           | —                                 | This workflow automatically processes Jotform survey responses and generates reports.                   |
| Sticky Note               | Sticky Note                      | Documentation and instructions                  | —                                 | —                                 | See section 5 for full sticky note content and links.                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Form Submission Trigger" Node**  
   - Type: Jotform Trigger  
   - Configure webhook with your Jotform form ID  
   - Ensure webhook is active in Jotform  

2. **Add "Collect Individual Response" Node**  
   - Type: Aggregate  
   - Set to aggregate all incoming data items (default)  
   - Connect output of Form Submission Trigger to this node  

3. **Add "Gemini LLM (Personal)" Node**  
   - Type: Google Gemini LLM (lmChatGoogleGemini)  
   - Use default options and link with n8n credentials for Google Gemini API  

4. **Add "Generate Personal Report" Node**  
   - Type: Langchain Agent (@n8n/n8n-nodes-langchain.agent)  
   - Set prompt text to instruct AI to extract demographics, analyze responses, and generate a concise HTML report  
   - Include system message to enforce style and output constraints  
   - Enable output parser (JSON only)  
   - Connect "Collect Individual Response" to "Generate Personal Report" input  
   - Connect "Gemini LLM (Personal)" node to the AI model input of this agent node  

5. **Add "Parse Personal Report JSON" Node**  
   - Type: Langchain Output Parser (Structured)  
   - Provide JSON schema example with "email_subject" and "html_report" fields  
   - Connect "Generate Personal Report" output to this node’s input  

6. **Add "Response Ready - Pause" Node**  
   - Type: Wait node  
   - Leave default settings (no delay)  
   - Connect "Parse Personal Report JSON" output to this node  

7. **Add "Send Personal Report" Node**  
   - Type: Gmail node  
   - Set recipient email dynamically using expression:  
     `={{ $('Form Submission Trigger').item.json['Please indicate your email so that we can reach you about the results.'] }}`  
   - Set subject to `={{ $json.output.email_subject }}`  
   - Set message body to `={{ $json.output.html_report }}`  
   - Disable attribution in options  
   - Connect "Response Ready - Pause" output to this node  

8. **Create "Weekly Report Scheduler" Node**  
   - Type: Schedule Trigger  
   - Set interval to weekly, trigger at 11:00  

9. **Add "Fetch Survey Submissions" Node**  
   - Type: HTTP Request  
   - Set URL to `https://api.jotform.com/form/YOUR_JOTFORM_FORM_ID/submissions`  
   - Add query parameter `apiKey` with your Jotform API key  
   - Connect "Weekly Report Scheduler" output to this node  

10. **Add "Unpack Response Objects" Node**  
    - Type: Split Out  
    - Set field to split on: `content`  
    - Include fields: `content.id, content.answers`  
    - Connect "Fetch Survey Submissions" output to this node  

11. **Add "Normalize Response Fields" Node**  
    - Type: Set  
    - Assign `content.id` to `={{ $json.content.id }}`  
    - Assign `content.answers` to `={{ $json.content.answers }}`  
    - Connect "Unpack Response Objects" output to this node  

12. **Add "Extract Q&A Pairs" Node**  
    - Type: Code  
    - Use provided JavaScript code to extract valid question-answer pairs filtering out controls  
    - Connect "Normalize Response Fields" output to this node  

13. **Add "Merge All Responses" Node**  
    - Type: Aggregate  
    - Aggregate all incoming items  
    - Connect "Extract Q&A Pairs" output to this node  

14. **Add "Gemini LLM" Node**  
    - Type: Google Gemini LLM  
    - Use same credentials as personal Gemini node  
    - Connect output to "Analyze & Report" AI model input  

15. **Add "Analyze & Report" Node**  
    - Type: Langchain Agent  
    - Set prompt instructing AI to generate aggregate report with stats, insights, and formatted HTML  
    - Provide system message emphasizing JSON output with subject and html_report  
    - Enable output parser  
    - Connect "Merge All Responses" output to this node  

16. **Add "Structured Output Parser" Node**  
    - Type: Langchain Output Parser (Structured)  
    - Use JSON schema example with "email_subject" and "html_report"  
    - Connect "Analyze & Report" output to this node  

17. **Add "Send Report to Admin" Node**  
    - Type: Gmail  
    - Set recipient to your admin email (e.g., YOUR_ADMIN_EMAIL@example.com)  
    - Set subject and message body from parsed AI fields  
    - Disable attribution  
    - Connect "Structured Output Parser" output to this node  

18. **Add Sticky Note for Documentation**  
    - Add a sticky note node with the workflow’s overview, instructions, and relevant links  

19. **Credentials Setup**  
    - Configure Google Gemini API credentials for LLM nodes  
    - Configure Gmail OAuth2 credentials for email nodes  
    - Ensure Jotform API key and form ID are correctly set in HTTP Request and webhook nodes  

20. **Testing**  
    - Submit a test survey response to verify personal report generation and email delivery  
    - Trigger the schedule manually or wait for weekly trigger to verify aggregate report generation and admin email delivery  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                      | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow automatically processes Jotform survey responses and generates two report types: personalized immediate feedback to respondents and weekly aggregated statistical reports to administrators.                                                                                                                                                       | Sticky Note within workflow                                                                       |
| Get started by creating surveys on [Jotform](https://www.jotform.com/?partner=roshanramanidev), obtain API keys, set up Google Gemini API credentials, and configure Gmail for sending emails.                                                                                                                                                                   | https://www.jotform.com/?partner=roshanramanidev                                                |
| The personalized reports focus on demographics, consumption habits, spending, health, and brand preference, returning a clean, mobile-friendly HTML email. Aggregate reports include respondent counts, statistical analyses, percentages, and key insights in professional HTML format.                                                                            | System messages guiding Langchain Agent nodes                                                   |
| Ensure credentials for Google Gemini and Gmail are valid and have appropriate API access and scopes for email sending and AI calls.                                                                                                                                                                                                                            | Credential setup notes                                                                          |
| The workflow uses advanced expressions to dynamically obtain respondent emails and inject data into AI prompts, requiring careful validation of survey field names and data formats.                                                                                                                                                                            | Expression usage in Send Personal Report node                                                  |
| The aggregate reporting block includes a custom JavaScript code node to parse and filter survey data, requiring maintenance if survey structure changes.                                                                                                                                                                                                     | Extract Q&A Pairs code node                                                                     |
| No raw JSON or markdown is included in AI outputs; strict JSON parsing is enforced to ensure data integrity before sending emails.                                                                                                                                                                                                                              | Output parsers configuration                                                                   |

---

**Disclaimer:** The provided text is exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly adheres to current content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.