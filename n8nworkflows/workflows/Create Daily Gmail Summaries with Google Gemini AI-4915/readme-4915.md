Create Daily Gmail Summaries with Google Gemini AI

https://n8nworkflows.xyz/workflows/create-daily-gmail-summaries-with-google-gemini-ai-4915


# Create Daily Gmail Summaries with Google Gemini AI

### 1. Workflow Overview

This workflow automates the creation of daily Gmail summaries using Google Gemini AI. Each day at a scheduled hour, it fetches all Gmail emails received since the previous day, processes them with an AI agent to generate a concise summary and extract important emails, and then refines the AI output for structured, reliable results. The workflow is designed for users who want to receive automated, AI-generated email digests highlighting key updates and urgent messages without manually sifting through their inbox.

The workflow is logically divided into the following blocks:

- **1.1 Scheduling & Date Calculation**: Triggering the workflow daily and computing the relevant date range for email retrieval.
- **1.2 Gmail Email Retrieval**: Fetching all emails received since the calculated date.
- **1.3 AI Processing & Summarization**: Feeding the collected emails into an AI agent configured with Google Gemini to generate summaries and identify important emails.
- **1.4 AI Output Parsing & Auto-Correction**: Structuring and validating the AI output, using an auto-fixing parser to ensure compliance with expected formats.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduling & Date Calculation

**Overview:**  
This block triggers the workflow every day at 8:00 AM and calculates the date exactly one day before the current date to define the email retrieval period.

**Nodes Involved:**  
- Schedule Trigger  
- Date & Time  
- Date & Time1

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Initiates workflow execution daily at a specified time.  
  - *Configuration:* Triggers once daily at 8:00 AM (triggerAtHour=8).  
  - *Input/Output:* No input; outputs a trigger event to "Date & Time".  
  - *Edge Cases:* Workflow won’t run if n8n instance is offline or the schedule is disabled.

- **Date & Time**  
  - *Type:* Date & Time  
  - *Role:* Captures the current date with time excluded.  
  - *Configuration:* Returns current date with time stripped (`includeTime=false`).  
  - *Input:* Trigger event from "Schedule Trigger".  
  - *Output:* Current date without time to "Date & Time1".  
  - *Edge Cases:* Timezone differences could affect date accuracy if n8n server timezone differs from user expectations.

- **Date & Time1**  
  - *Type:* Date & Time  
  - *Role:* Calculates the date one day prior to the current date for filtering emails.  
  - *Configuration:* Subtracts 1 day from the date supplied by "Date & Time".  
  - *Input:* Current date from "Date & Time".  
  - *Output:* Previous day’s date passed as `receivedAfter` filter to "Gmail".  
  - *Edge Cases:* Same timezone considerations apply; subtraction operation must handle date boundaries correctly.

---

#### 1.2 Gmail Email Retrieval

**Overview:**  
This block fetches all Gmail emails received since the calculated previous day date.

**Nodes Involved:**  
- Gmail

**Node Details:**

- **Gmail**  
  - *Type:* Gmail node (OAuth2)  
  - *Role:* Retrieves all emails filtered by read status and received date.  
  - *Configuration:*  
    - Operation: `getAll` emails.  
    - Filter: Emails with any read status (`both`) received after the date from "Date & Time1".  
  - *Input:* Date filter from "Date & Time1".  
  - *Output:* JSON array containing all matching emails to "AI Agent".  
  - *Credentials:* Uses OAuth2 Gmail account authentication.  
  - *Edge Cases:*  
    - API limits or quota exceeded errors.  
    - Network or auth token expiration errors.  
    - Large email volumes could cause timeouts or slow performance.  
    - Emails without bodies or malformed data.

---

#### 1.3 AI Processing & Summarization

**Overview:**  
This block sends the retrieved emails to an AI agent that summarizes the daily emails and identifies important or time-sensitive messages.

**Nodes Involved:**  
- AI Agent  
- Google Gemini Chat Model

**Node Details:**

- **AI Agent**  
  - *Type:* Langchain Agent (AI Processor)  
  - *Role:* Processes raw email data and generates a textual summary and importance identification.  
  - *Configuration:*  
    - Input Text: JSON stringified array of all emails from "Gmail" node.  
    - System Message: Defines AI’s task as an intelligent email assistant to summarize emails and identify important ones with subject, sender, and reason.  
    - Prompt Type: `define` (custom defined prompt).  
    - Output Parsing: Enabled (`hasOutputParser=true`).  
    - Execute Once: true (ensures single execution per input).  
  - *Input:* Emails JSON from "Gmail".  
  - *Output:* Raw AI-generated text to "Google Gemini Chat Model" as language model.  
  - *Edge Cases:*  
    - Input JSON too large or malformed causing parsing failures.  
    - AI model errors or rate limits.  
    - Ambiguous or incomplete email data affecting summary relevance.

- **Google Gemini Chat Model**  
  - *Type:* Langchain Google Gemini Chat Model  
  - *Role:* Provides the actual AI language model inference using Google Gemini.  
  - *Configuration:*  
    - Model: `models/gemini-2.0-flash`.  
    - Temperature: 0.2 (low randomness for more precise output).  
  - *Credentials:* Google PaLM API credentials.  
  - *Input:* AI Agent prompt text.  
  - *Output:* AI-generated response text to "Auto-fixing Output Parser".  
  - *Edge Cases:*  
    - API quota exceeded or authentication errors.  
    - Latency or timeout issues.  
    - Model version changes affecting output format.

---

#### 1.4 AI Output Parsing & Auto-Correction

**Overview:**  
This block ensures the AI output is parsed into a structured JSON format and auto-corrects it if it does not satisfy the expected schema or constraints.

**Nodes Involved:**  
- Auto-fixing Output Parser  
- Structured Output Parser

**Node Details:**

- **Auto-fixing Output Parser**  
  - *Type:* Langchain Output Parser Autofixing  
  - *Role:* Validates AI output against instructions; if parsing fails, requests re-generation with corrections.  
  - *Configuration:* Uses a prompt template that includes instructions, completion, and error details, asking the AI to retry with compliant output only.  
  - *Input:* Raw AI text from "Google Gemini Chat Model".  
  - *Output:* Corrected AI output JSON to "Structured Output Parser".  
  - *Edge Cases:*  
    - Infinite correction loops if AI repeatedly fails to comply.  
    - Parsing errors if output structure is severely malformed.

- **Structured Output Parser**  
  - *Type:* Langchain Output Parser Structured  
  - *Role:* Parses final AI output into structured JSON based on example schema.  
  - *Configuration:* Provided a JSON schema example illustrating expected summary and important emails format.  
  - *Input:* Corrected output from "Auto-fixing Output Parser".  
  - *Output:* Final structured JSON summary for use or downstream processing.  
  - *Edge Cases:* Schema mismatch or missing fields causing parse failures.

---

### 3. Summary Table

| Node Name               | Node Type                              | Functional Role                        | Input Node(s)         | Output Node(s)             | Sticky Note                                                                                                                   |
|-------------------------|--------------------------------------|-------------------------------------|-----------------------|----------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger                     | Initiates workflow daily at 8:00 AM | -                     | Date & Time                |                                                                                                                              |
| Date & Time             | Date & Time                         | Gets current date without time       | Schedule Trigger       | Date & Time1               |                                                                                                                              |
| Date & Time1            | Date & Time                         | Calculates previous day’s date       | Date & Time            | Gmail                      |                                                                                                                              |
| Gmail                   | Gmail                               | Retrieves all emails since previous day | Date & Time1          | AI Agent                   |                                                                                                                              |
| AI Agent                | Langchain Agent                     | Analyzes emails, generates summary and important emails | Gmail                  | Google Gemini Chat Model    |                                                                                                                              |
| Google Gemini Chat Model | Langchain Google Gemini Chat Model | Performs AI language model inference | AI Agent               | Auto-fixing Output Parser  |                                                                                                                              |
| Auto-fixing Output Parser| Langchain Output Parser Autofixing | Validates and auto-corrects AI output | Google Gemini Chat Model | Structured Output Parser   |                                                                                                                              |
| Structured Output Parser | Langchain Output Parser Structured  | Parses final AI output into structured JSON | Auto-fixing Output Parser | -                          | Example JSON schema included for expected output format.                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Schedule Trigger" node**  
   - Type: Schedule Trigger  
   - Parameters: Set to trigger daily at 8:00 AM (triggerAtHour=8).  
   - No credentials needed.

2. **Create "Date & Time" node**  
   - Type: Date & Time  
   - Parameters: Enable `includeTime` = false (to get date only).  
   - Connect input from "Schedule Trigger".

3. **Create "Date & Time1" node**  
   - Type: Date & Time  
   - Parameters:  
     - Operation: Subtract from date  
     - Duration: 1  
     - Magnitude: Use expression referencing current date from "Date & Time" node (`={{ $json.currentDate }}`).  
   - Connect input from "Date & Time".

4. **Create "Gmail" node**  
   - Type: Gmail  
   - Parameters:  
     - Operation: `getAll`  
     - Filters:  
       - `readStatus`: both (includes read and unread)  
       - `receivedAfter`: Expression referencing output from "Date & Time1" node (`={{ $json.newDate }}`)  
   - Credentials: Configure with valid Gmail OAuth2 credentials.  
   - Connect input from "Date & Time1".

5. **Create "AI Agent" node**  
   - Type: Langchain Agent  
   - Parameters:  
     - Text: Expression `={{ $('Gmail').all().toJsonString() }}` to stringify all fetched emails.  
     - System Message:  
       ```
       You are an intelligent email assistant. Your task is to analyze a list of email messages received in the last 24 hours. Each email includes a subject, sender, timestamp, and body.

       Your goals are:
         1. Summary: Provide a concise summary of the overall themes or key updates across all emails.
         2. Important Emails: Identify and list the most important or time-sensitive emails. For each, include:
           • Subject
           • Sender
           • Reason why it’s important (e.g., deadline, meeting, urgent request, opportunity)
       ```  
     - Prompt Type: define  
     - Enable Output Parser: true  
     - Execute Once: true  
   - Connect input from "Gmail".

6. **Create "Google Gemini Chat Model" node**  
   - Type: Langchain Google Gemini Chat Model  
   - Parameters:  
     - Model Name: `models/gemini-2.0-flash`  
     - Temperature: 0.2  
   - Credentials: Add Google PaLM API credentials.  
   - Connect AI language model input from "AI Agent".

7. **Create "Auto-fixing Output Parser" node**  
   - Type: Langchain Output Parser Autofixing  
   - Parameters:  
     - Prompt:  
       ```
       Instructions:
       --------------
       {instructions}
       --------------
       Completion:
       --------------
       {completion}
       --------------

       Above, the Completion did not satisfy the constraints given in the Instructions.
       Error:
       --------------
       {error}
       --------------

       Please try again. Please only respond with an answer that satisfies the constraints laid out in the Instructions:
       ```  
   - Connect AI output parser input from "Google Gemini Chat Model".

8. **Create "Structured Output Parser" node**  
   - Type: Langchain Output Parser Structured  
   - Parameters:  
     - JSON Schema Example:  
       ```
       {
         "summary": "Today’s emails covered project updates, meeting schedules, and one urgent request.\n Important Emails: \n 1. subject :Client Meeting Rescheduled , sender : test@test.com , reason : The meeting has been moved to tomorrow morning, requiring immediate schedule adjustment.\n 2. subject :Client Meeting Rescheduled , sender : test@test.com , reason : The meeting has been moved to tomorrow morning, requiring immediate schedule adjustment."
       }
       ```  
   - Connect AI output parser input from "Auto-fixing Output Parser".

9. **Set Execution Order**  
   - Ensure node connections follow:  
     Schedule Trigger → Date & Time → Date & Time1 → Gmail → AI Agent → Google Gemini Chat Model → Auto-fixing Output Parser → Structured Output Parser

10. **Validate Credentials**  
    - Gmail OAuth2: Ensure OAuth2 credentials are valid and authorized for Gmail API scopes.  
    - Google PaLM API: Configure API key or OAuth2 for Google Gemini AI access.

11. **Test Workflow**  
    - Manually trigger to verify email retrieval, AI processing, and output parsing.  
    - Monitor for errors such as API rate limits, parsing errors, or empty email sets.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                 |
|----------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| Workflow uses Google Gemini AI (PaLM API) for advanced email summarization and important email detection. | Requires Google PaLM API credentials.                          |
| Gmail node uses OAuth2 for secure access; tokens must be refreshed periodically.                     | OAuth2 Gmail setup within n8n.                                 |
| AI Output Parser includes an auto-fixing mechanism to improve robustness against AI output format variations. | Helps avoid failures due to unexpected AI responses.          |
| Example JSON schema in Structured Output Parser guides expected AI output format for downstream use. | Ensures consistent structured data for integration or notifications. |

---

**Disclaimer:** The provided content is extracted exclusively from an n8n automated workflow and complies with current content policies. It contains no illegal, offensive, or protected elements. All data processed is lawful and publicly accessible.