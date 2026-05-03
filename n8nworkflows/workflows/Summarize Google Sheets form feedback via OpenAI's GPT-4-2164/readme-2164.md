Summarize Google Sheets form feedback via OpenAI's GPT-4

https://n8nworkflows.xyz/workflows/summarize-google-sheets-form-feedback-via-openai-s-gpt-4-2164


# Summarize Google Sheets form feedback via OpenAI's GPT-4

### 1. Workflow Overview

This n8n workflow is designed to automate the process of summarizing event feedback collected via a Google Form and stored in a Google Sheets document. The primary use case is to streamline feedback analysis by aggregating responses, leveraging OpenAI's GPT-4 model to generate a summary report, converting that report to HTML, and emailing it to a predefined recipient.

The workflow is logically structured into four main blocks:

- **1.1 Input Reception:** Manual trigger initiation and retrieval of Google Sheets records containing form responses.
- **1.2 Data Aggregation:** Combining multiple feedback entries per question into arrays to facilitate comprehensive analysis.
- **1.3 AI Processing:** Using OpenAI's GPT-4 to analyze aggregated feedback and generate a detailed summary report in Markdown.
- **1.4 Output and Notification:** Converting the Markdown summary to HTML and sending it via email through Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block initiates the workflow manually and fetches all new feedback entries from the specified Google Sheets document linked to the feedback form.

**Nodes Involved:**  
- When clicking "Test workflow"  
- Get Google Sheets records  

**Node Details:**

- **When clicking "Test workflow"**  
  - *Type:* Manual Trigger  
  - *Role:* Starts the workflow manually on demand, suitable for testing or on-demand execution.  
  - *Configuration:* No parameters, simply a manual trigger node.  
  - *Inputs:* None (start node)  
  - *Outputs:* Connects to "Get Google Sheets records"  
  - *Edge cases:* No automated scheduling; relies on manual initiation.  

- **Get Google Sheets records**  
  - *Type:* Google Sheets (v4.2)  
  - *Role:* Retrieves all rows from the Google Sheets document where form responses are stored.  
  - *Configuration:*  
    - Document ID set to the Google Sheets file linked to the feedback form.  
    - Sheet Name targeting the specific sheet tab "Form Responses 1" (ID: 2035968519).  
    - Uses OAuth2 credentials for Google account authentication ("Ted's Tech Talks Google account").  
  - *Inputs:* From manual trigger node  
  - *Outputs:* Feeds data to "Aggregate responses into arrays"  
  - *Edge cases:*  
    - Authentication failures if OAuth token expires or permissions are revoked.  
    - Empty or malformed spreadsheet data.  
    - Large datasets may hit Google Sheets API limits or timeouts.  

#### 1.2 Data Aggregation

**Overview:**  
This block aggregates all individual responses per question into arrays, producing a single item that contains all feedback grouped by question to simplify subsequent analysis.

**Nodes Involved:**  
- Aggregate responses into arrays  

**Node Details:**

- **Aggregate responses into arrays**  
  - *Type:* Aggregate  
  - *Role:* Collapses multiple rows of responses into a single JSON object with arrays for each question.  
  - *Configuration:*  
    - Aggregates fields for the three main questions:  
      1. "What went great?"  
      2. "How can we improve?"  
      3. "What is the chance of recommending our event?"  
    - Each field is aggregated into an array of corresponding answers.  
  - *Inputs:* From "Get Google Sheets records"  
  - *Outputs:* Passes aggregated data to "Summarize via GPT model"  
  - *Edge cases:*  
    - Missing expected fields in the sheet may cause empty arrays or errors.  
    - If new questions are added, the node configuration must be updated accordingly.  

#### 1.3 AI Processing

**Overview:**  
This block sends the aggregated feedback to OpenAI's GPT-4 model to generate a comprehensive summary, including sentiment analysis and improvement suggestions, formatted in Markdown.

**Nodes Involved:**  
- Summarize via GPT model  

**Node Details:**

- **Summarize via GPT model**  
  - *Type:* OpenAI (Chat Completion API)  
  - *Role:* Processes aggregated feedback and returns a Markdown-formatted summary report.  
  - *Configuration:*  
    - Model: GPT-4 Turbo Preview (chat resource).  
    - Temperature set to 0.3 for focused and consistent output.  
    - Prompt consists of two messages:  
      - System message instructs the model to analyze the three questions and prepare a summary report including overall sentiment and constructive suggestions.  
      - User message dynamically inserts concatenated feedback arrays per question, separated by “|” characters, using expressions such as:  
        ```  
        1. What went great: ```{{ $json['What went great?'].join(' | ') }}```  
        2. How can we improve: ```{{ $json['How can we improve?'].join(' | ') }}```  
        3. What is the chance of recommending our event: ```{{ $json['What is the chance of recommending our event?'].join(' | ') }}```  
        ```  
  - *Inputs:* From "Aggregate responses into arrays"  
  - *Outputs:* Sends the Markdown summary to "Convet from Markdown to HTML"  
  - *Edge cases:*  
    - OpenAI API rate limits, quota exhaustion, or authentication errors.  
    - Very large input arrays may exceed token limits; the note suggests splitting large forms or datasets.  
    - Potential expression resolution failures if input data structure changes.  

#### 1.4 Output and Notification

**Overview:**  
This block converts the AI-generated Markdown summary into HTML and sends it via Gmail to the specified email address.

**Nodes Involved:**  
- Convet from Markdown to HTML (note: typo in node name)  
- Send via Gmail  

**Node Details:**

- **Convet from Markdown to HTML**  
  - *Type:* Markdown  
  - *Role:* Converts the Markdown summary report into HTML format for email compatibility.  
  - *Configuration:*  
    - Mode: markdownToHtml  
    - Option: Does not output a complete HTML document, only the converted snippet.  
    - Markdown input expression: `={{ $json.message.content }}` (extracts the content from the OpenAI node output).  
  - *Inputs:* From "Summarize via GPT model"  
  - *Outputs:* To "Send via Gmail"  
  - *Edge cases:*  
    - Markdown syntax errors could lead to malformed HTML.  
    - If the OpenAI output is missing or malformed, conversion will fail.  

- **Send via Gmail**  
  - *Type:* Gmail (v2.1)  
  - *Role:* Sends an email with the generated HTML summary to a predefined recipient.  
  - *Configuration:*  
    - Recipient email hardcoded as "teds.tech.talks@gmail.com".  
    - Message body is set to the HTML content from the previous node.  
    - Subject: "Feedback form response".  
    - Attribution disabled to avoid adding extra text.  
    - Uses OAuth2 credentials named "Gmail account".  
  - *Inputs:* From "Convet from Markdown to HTML"  
  - *Outputs:* None (end node)  
  - *Edge cases:*  
    - Authentication failures or expired tokens.  
    - Email delivery issues due to wrong recipient or SMTP problems.  

---

### 3. Summary Table

| Node Name                    | Node Type          | Functional Role                                | Input Node(s)                 | Output Node(s)                  | Sticky Note                                                                                                  |
|------------------------------|--------------------|-----------------------------------------------|------------------------------|---------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking "Test workflow" | Manual Trigger     | Initiates the workflow manually                | None                         | Get Google Sheets records       | [Link to the Google Sheets template](https://docs.google.com/spreadsheets/d/1Kcr1oF_RrfNQJczmJDpwClOSYpvSnwbeX-_pdUo91-I/edit?usp=sharing) <br> Create a Google Sheet document. |
| Get Google Sheets records     | Google Sheets      | Retrieves all form response records            | When clicking "Test workflow" | Aggregate responses into arrays |                                                                                                              |
| Aggregate responses into arrays | Aggregate         | Combines responses per question into arrays   | Get Google Sheets records     | Summarize via GPT model         | Combine all answers into an array to prepare data for analysis.                                              |
| Summarize via GPT model       | OpenAI Chat        | Generates summary report from aggregated data | Aggregate responses into arrays | Convet from Markdown to HTML     | Generate a summary report with system and user messages. <br> Note on splitting large forms or datasets.    |
| Convet from Markdown to HTML  | Markdown           | Converts Markdown summary to HTML              | Summarize via GPT model       | Send via Gmail                  | Converts GPT reply in Markdown to HTML for email.                                                           |
| Send via Gmail                | Gmail              | Sends the HTML-formatted summary via email    | Convet from Markdown to HTML  | None                           | Sends email to "teds.tech.talks@gmail.com".                                                                  |
| Sticky Note                  | Sticky Note        | Documentation and guidance                      | None                         | None                           | See notes above for each sticky note covering distinct workflow steps.                                       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Add a **Manual Trigger** node named **When clicking "Test workflow"**.  
   - No parameters needed.  

2. **Add Google Sheets Node**  
   - Add a **Google Sheets** node named **Get Google Sheets records**.  
   - Set **Authentication** to OAuth2 credential for your Google account.  
   - Set **Document ID** to your Google Sheets file containing form responses.  
   - Select the appropriate **Sheet Name** (e.g., "Form Responses 1").  
   - Connect output of **When clicking "Test workflow"** to this node.  

3. **Add Aggregate Node**  
   - Add an **Aggregate** node named **Aggregate responses into arrays**.  
   - Under **Fields to Aggregate**, add each question field exactly as in the sheet:  
     - "What went great?"  
     - "How can we improve?"  
     - "What is the chance of recommending our event?"  
   - Set aggregation mode to combine values into arrays (default).  
   - Connect output of **Get Google Sheets records** to this node.  

4. **Add OpenAI Node**  
   - Add an **OpenAI** node named **Summarize via GPT model**.  
   - Set **Resource** to "Chat" and **Operation** to "Chat Completion".  
   - Choose **Model**: `gpt-4-turbo-preview`.  
   - Set **Temperature** to 0.3 for controlled randomness.  
   - Add two prompt messages:  
     - **System message:**  
       ```
       Your task is to summarize event feedback form responses. You will receive answers on three questions:
       1. What went great?
       2. How can we improve?
       3. What is the chance of recommending our event?

       Each questions has several answers separated by | character.
       Analyze each question and prepare a summary report. It should contain an overall sentiment regarding the event, followed by the constructive ideas of what to improve.

       Reply in Markdown formatting
       ```  
     - **User message:**  
       ```
       =1. What went great: ```{{ $json['What went great?'].join(' | ') }}```
       2. How can we improve: ```{{ $json['How can we improve?'].join(' | ') }}```
       3. What is the chance of recommending our event: ```{{ $json['What is the chance of recommending our event?'].join(' | ') }}```
       ```  
   - Use OpenAI API credentials linked to your account.  
   - Connect output of **Aggregate responses into arrays** to this node.  

5. **Add Markdown Node**  
   - Add a **Markdown** node named **Convet from Markdown to HTML** (note: name typo is optional).  
   - Set **Mode** to `markdownToHtml`.  
   - Disable "Complete HTML Document" option.  
   - Set **Markdown** field to expression: `={{ $json.message.content }}` to pull the GPT response content.  
   - Connect output of **Summarize via GPT model** to this node.  

6. **Add Gmail Node**  
   - Add a **Gmail** node named **Send via Gmail**.  
   - Configure **Send To** with the email address `teds.tech.talks@gmail.com` or your desired recipient.  
   - Set **Subject** to `"Feedback form response"`.  
   - Use expression `={{ $json.data }}` for the message body to insert HTML content.  
   - Disable attribution to avoid extra text.  
   - Authenticate using Gmail OAuth2 credentials.  
   - Connect output of **Convet from Markdown to HTML** to this node.  

7. **Save and Test**  
   - Save the workflow.  
   - Trigger manually via **When clicking "Test workflow"** node.  
   - Verify email receipt and content formatting.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| This workflow uses a Google Sheets template linked to a Google Form for event feedback collection. Adapt the trigger as needed for your use case (manual or scheduled).                                                           | [Google Sheets template](https://docs.google.com/spreadsheets/d/1Kcr1oF_RrfNQJczmJDpwClOSYpvSnwbeX-_pdUo91-I/edit?usp=sharing)   |
| For large forms or high volumes of responses, consider splitting input data before sending to GPT to avoid exceeding token limits.                                                                                              | Workflow sticky note within AI Processing block                                                                                   |
| GPT output is generated in Markdown and converted to HTML for email compatibility.                                                                                                                                              | Conversion occurs in Markdown node                                                                                                |
| Ensure OAuth2 credentials for Google Sheets and Gmail are correctly set up and authorized with required scopes.                                                                                                                  | Critical for authentication; token expiry or permission revocation will cause failures                                          |
| The workflow is tagged under "Ted's Tech Talks," indicating project or organizational context.                                                                                                                                   | Useful for tracking and categorizing workflows                                                                                   |

---

This comprehensive documentation should enable advanced users or automation agents to fully understand, reproduce, and modify the workflow efficiently while anticipating common issues or edge cases.