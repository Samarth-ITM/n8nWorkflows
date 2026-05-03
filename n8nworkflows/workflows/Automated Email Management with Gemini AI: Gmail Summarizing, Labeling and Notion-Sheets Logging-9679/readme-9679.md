Automated Email Management with Gemini AI: Gmail Summarizing, Labeling and Notion/Sheets Logging

https://n8nworkflows.xyz/workflows/automated-email-management-with-gemini-ai--gmail-summarizing--labeling-and-notion-sheets-logging-9679


# Automated Email Management with Gemini AI: Gmail Summarizing, Labeling and Notion/Sheets Logging

### 1. Workflow Overview

This workflow automates email management for Gmail using AI-driven analysis and labeling, complemented by logging email summaries into Notion and Google Sheets. It targets users who want to streamline their inbox by automatically summarizing, categorizing, and tracking emails with minimal manual effort. The workflow is structured into the following functional blocks:

- **1.1 Input Reception:** Triggering on new incoming Gmail messages.
- **1.2 Email Data Extraction:** Processing and formatting raw email data for AI consumption.
- **1.3 AI-Powered Email Analysis:** Leveraging a large language model (Google Gemini via LangChain) to summarize the email and suggest or create appropriate labels.
- **1.4 Label Management:** Creating new Gmail labels if needed, retrieving label IDs, and applying labels to emails.
- **1.5 Logging:** Storing processed email summaries and metadata into Notion and Google Sheets for record-keeping and further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the entire workflow when a new email arrives in the Gmail inbox and initiates the email processing pipeline.

**Nodes Involved:**  
- On new Email

**Node Details:**  
- **On new Email**  
  - Type: Gmail Trigger  
  - Configuration: Watches the Gmail inbox label "INBOX" for new messages, polling every hour at minute 59. Attachments are not downloaded.  
  - Input: None (external event trigger)  
  - Output: Raw email data JSON from Gmail  
  - Version Requirement: Compatible with Gmail OAuth2 credentials.  
  - Possible Failures: Authentication errors, polling delays, API rate limits.

---

#### 2.2 Email Data Extraction

**Overview:**  
Processes raw Gmail data to extract sender information, subject, content (in plain text), date, and attachment presence. It formats these fields into a consistent JSON structure for AI analysis and downstream use.

**Nodes Involved:**  
- Process Email Data

**Node Details:**  
- **Process Email Data**  
  - Type: Code (JavaScript) node for data transformation  
  - Configuration:  
    - Extracts sender name and email using multiple strategies to handle different email header formats.  
    - Formats the received date into US locale string with date and time.  
    - Extracts email content prioritizing plain text; falls back to HTML after stripping tags and CSS or other fields.  
    - Truncates content to 5000 characters to prevent processing overload.  
    - Outputs a structured object containing messageId, threadId, sender details, subject, content, date, and attachment flag.  
  - Key Expressions: Uses `$input.all()` to process all incoming emails, date formatting with `toLocaleString`, regex for email/name parsing.  
  - Input: Raw Gmail email JSON  
  - Output: Structured email data JSON  
  - Edge Cases: Emails without standard headers, missing fields, very long content, HTML emails with complex formatting, possible failure if date is missing or malformed.

---

#### 2.3 AI-Powered Email Analysis

**Overview:**  
Analyzes the extracted email data with an AI agent to produce a detailed summary and suggest a suitable Gmail label category. It also indicates if a new label should be created and flags priority and action requirements.

**Nodes Involved:**  
- AI Email Analyzer  
- Structured Output Parser  
- Google Gemini Chat Model (as AI language model) (optional integration)

**Node Details:**  
- **AI Email Analyzer**  
  - Type: LangChain Agent node  
  - Configuration:  
    - Uses a prompt instructing the AI to summarize the email and select or suggest labels from a predefined list or create a new label if none fit.  
    - Input variables include sender name/email, subject, email content, and attachment presence from the processed email data.  
    - Output expected in a strict JSON format with keys: summary, suggestedLabel, createNewLabel, priority, actionRequired, keyPoints.  
  - Input: Process Email Data output  
  - Output: AI response with email summary and labeling recommendation  
  - Edge Cases: AI may produce malformed JSON, incomplete analysis, or mislabel; integration failures if AI service is down or API quota exceeded.

- **Structured Output Parser**  
  - Type: LangChain structured output parser  
  - Configuration: Validates and parses the AI response JSON, enforcing schema for messageId, threadId, summary, and emailLabel.  
  - Input: Raw AI response JSON  
  - Output: Clean, validated structured output for label creation and further steps  
  - Edge Cases: Parsing errors if AI output deviates from schema.

- **Google Gemini Chat Model**  
  - Type: LangChain AI language model node  
  - Configuration: Uses Google’s Gemini (PaLM) API as the underlying LLM for AI Email Analyzer and Label Agent nodes.  
  - Credentials: Google Palm API key provided  
  - Edge Cases: API failures, latency, or quota issues.

---

#### 2.4 Label Management

**Overview:**  
Manages Gmail labels by creating new labels suggested by the AI, retrieving existing label IDs, and applying the correct label to the email.

**Nodes Involved:**  
- Create Gmail Label  
- Get Label  
- Label Agent  
- Add Label to Email

**Node Details:**  
- **Create Gmail Label**  
  - Type: Gmail Node (Label resource)  
  - Configuration: Creates a Gmail label using the label name suggested by the AI.  
  - Continues workflow on failure (e.g., label already exists).  
  - Input: Label name from AI Email Analyzer's output.  
  - Output: Confirmation or error info for label creation.  
  - Edge Cases: Label name conflicts, API quota exceeded, permission errors.

- **Get Label**  
  - Type: Gmail Tool Node (Label resource)  
  - Configuration: Retrieves all existing Gmail labels with their IDs.  
  - Input: Triggered after label creation step.  
  - Output: List of label objects with names and IDs.  
  - Edge Cases: API rate limits, authentication errors.

- **Label Agent**  
  - Type: LangChain Agent node  
  - Configuration: Input is the label name suggested by AI Email Analyzer and the list of all Gmail labels fetched.  
  - Task: Match the suggested label name to a Gmail label ID and output only the label ID for application.  
  - Output: Label ID string.  
  - Edge Cases: Label not found in fetched list, multiple labels with similar names, AI misinterpretation.

- **Add Label to Email**  
  - Type: Gmail Node (Message resource)  
  - Configuration: Adds the label ID from Label Agent to the email message identified by messageId.  
  - Input: Label ID, messageId from Process Email Data.  
  - Output: Confirmation of label application.  
  - Edge Cases: Incorrect label ID format causing failure, messageId missing, Gmail API errors.

---

#### 2.5 Logging

**Overview:**  
Logs the email's summarized details, labels, and metadata into Notion and Google Sheets for record keeping and further analysis.

**Nodes Involved:**  
- Logs in Google Sheets  
- Logs in Notion

**Node Details:**  
- **Logs in Google Sheets**  
  - Type: Google Sheets Append Row operation  
  - Configuration: Writes columns such as Date, Label, Subject, Summary, Message ID, Sender Name, Sender Email into a specified Google Sheet.  
  - Inputs: Mixed data from Process Email Data and AI Email Analyzer.  
  - Credentials: Google Sheets OAuth2 account.  
  - Edge Cases: Sheet not found, permission issues, API quota, data type mismatches.

- **Logs in Notion**  
  - Type: Notion Create Database Page  
  - Configuration: Creates a new page in a specified Notion database with properties: Message ID, Sender Name, Sender Email, Subject, Summary, Date & Time, and Label (multi-select).  
  - Inputs: Data from Process Email Data and AI Email Analyzer.  
  - Credentials: Notion API token.  
  - Edge Cases: Database ID invalid, API permission errors, rate limits.

---

### 3. Summary Table

| Node Name           | Node Type                         | Functional Role                             | Input Node(s)         | Output Node(s)             | Sticky Note                                                                                             |
|---------------------|----------------------------------|---------------------------------------------|-----------------------|----------------------------|--------------------------------------------------------------------------------------------------------|
| On new Email        | Gmail Trigger                    | Triggers workflow on new Gmail messages     | None                  | Process Email Data         | # Gmail Trigger and Email Process: Triggers on every message (default). Polls every 59 mins.           |
| Process Email Data  | Code                            | Extracts and formats email data              | On new Email           | AI Email Analyzer          | # Gmail Trigger and Email Process: Takes out essential data from the email received                    |
| AI Email Analyzer   | LangChain Agent                 | Summarizes email and suggests label          | Process Email Data     | Create Gmail Label         | # Email Analyzer: Analyzes and summarizes email, suggests specific label                               |
| Structured Output Parser | LangChain Structured Output Parser | Parses AI JSON response into structured format | AI Email Analyzer      | AI Email Analyzer          |                                                                                                        |
| Create Gmail Label  | Gmail Label Create              | Creates suggested Gmail label                | AI Email Analyzer      | Label Agent                | # Creates Label: Creates label from AI suggestion; continues if label exists                            |
| Get Label           | Gmail Tool (Label)              | Retrieves all Gmail labels                    | Create Gmail Label     | Label Agent                |                                                                                                        |
| Label Agent         | LangChain Agent                 | Matches label name to Gmail label ID         | Get Label              | Add Label to Email         | # AI Label Assigner: Forwards appropriate label ID for email                                          |
| Add Label to Email  | Gmail Message Label Add         | Applies label ID to the email                 | Label Agent            | Logs in Google Sheets, Logs in Notion | # Add the Label: Only add Label ID or workflow will fail                                              |
| Logs in Google Sheets | Google Sheets Append Row       | Logs email data into Google Sheets            | Add Label to Email     | None                      | # Logging the details in G-Sheet and Notion                                                            |
| Logs in Notion      | Notion Create Database Page     | Logs email data into Notion database          | Add Label to Email     | None                      | # Logging the details in G-Sheet and Notion                                                            |
| Google Gemini Chat Model | LangChain Language Model    | Provides AI language model for analysis       | (Used by AI nodes)     | (Feeds AI nodes)           | # AI Node: You can choose your own favourite LLM                                                       |
| Sticky Note         | Sticky Note                    | Documentation and explanation notes           | None                  | None                      | Various nodes have sticky notes explaining block roles and usage                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Gmail Trigger Node ("On new Email")**  
   - Type: Gmail Trigger  
   - Configure to watch "INBOX" label only  
   - Set polling interval to every hour at minute 59  
   - Use valid Gmail OAuth2 credentials  

2. **Add a Code Node ("Process Email Data")**  
   - Purpose: Extract sender info, subject, content, date, attachments  
   - Paste provided JavaScript to parse the email JSON input  
   - Output structured JSON with keys: messageId, threadId, Date, Sender Name, Sender Email, Subject, Content, Has Attachments  

3. **Add a LangChain Agent Node ("AI Email Analyzer")**  
   - Use Google Gemini or preferred LLM as AI language model  
   - Set prompt to analyze email and produce summary and suggested label in specified JSON format  
   - Use input expressions to pass sender name/email, subject, content, attachment status from "Process Email Data" output  

4. **Add a LangChain Structured Output Parser Node ("Structured Output Parser")**  
   - Configure schema validation matching AI's JSON format  
   - Connect AI Email Analyzer's raw output to this node  
   - Outputs structured, validated JSON  

5. **Add a Gmail Node to Create Label ("Create Gmail Label")**  
   - Operation: Create label  
   - Label name: Use AI Email Analyzer's suggested label (expression from structured output)  
   - Set to continue on fail (to handle existing labels)  
   - Use Gmail OAuth2 credentials  

6. **Add Gmail Tool Node to Get Labels ("Get Label")**  
   - Operation: List all labels  
   - Use same Gmail credentials  
   - Ensures the workflow has the full label list with IDs  

7. **Add a LangChain Agent Node ("Label Agent")**  
   - Input: Suggested label name from AI output and the full label list from "Get Label"  
   - Task: Find matching label ID for suggested label, output only the label ID string  
   - Use Google Gemini or same AI language model  

8. **Add Gmail Node to Add Label to Email ("Add Label to Email")**  
   - Operation: Add label to message  
   - Label IDs: Output from "Label Agent"  
   - Message ID: From "Process Email Data"  
   - Use Gmail OAuth2 credentials  

9. **Add Google Sheets Node ("Logs in Google Sheets")**  
   - Operation: Append row to existing sheet  
   - Map columns: Date, Label, Subject, Summary, Message ID, Sender Name, Sender Email from respective nodes  
   - Authenticate with Google Sheets OAuth2  

10. **Add Notion Node ("Logs in Notion")**  
    - Operation: Create database page  
    - Database ID: Specify your Notion database for email summaries  
    - Map properties similarly to Google Sheets (Message ID, Sender Name, Sender Email, Subject, Summary, Date & Time, Label multi-select)  
    - Authenticate with Notion API credentials  

11. **Connect nodes in the order:**  
    - On new Email → Process Email Data → AI Email Analyzer → Create Gmail Label → Get Label → Label Agent → Add Label to Email → Logs in Google Sheets & Logs in Notion

12. **Optional:** Add sticky notes describing each block for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                       | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| The AI Email Analyzer uses Google Gemini (PaLM) via LangChain integration, which can be substituted with any other preferred LLM with adjustment. | AI Node integration                                                                              |
| Gmail API usage requires OAuth2 credentials with sufficient scope for reading labels, messages, and managing labels and messages.                 | Gmail OAuth2 setup instructions                                                                 |
| Notion database must have properties matching those mapped in the "Logs in Notion" node for successful logging.                                   | Notion API documentation                                                                        |
| Google Sheets must have a sheet with appropriate columns matching the mapped keys for logging to work correctly.                                  | Google Sheets API and OAuth2 configuration                                                      |
| The workflow gracefully handles label creation failures (e.g., if the label already exists) by continuing execution.                              | Label creation fault tolerance                                                                   |
| Email content truncation at 5000 characters avoids exceeding AI input size limits.                                                                | Email content processing limitation                                                             |
| Sticky notes in the workflow provide useful inline documentation on node roles and configurations.                                                | Helpful for maintenance and onboarding                                                          |
| Workflow assumes emails are mostly in English and the AI prompt and date formatting are tailored accordingly.                                     | Localization considerations                                                                      |

---

**Disclaimer:** The provided description and analysis originate exclusively from an automated n8n workflow. The content fully complies with all applicable policies and contains no illegal or protected material. All processed data is lawful and publicly accessible.