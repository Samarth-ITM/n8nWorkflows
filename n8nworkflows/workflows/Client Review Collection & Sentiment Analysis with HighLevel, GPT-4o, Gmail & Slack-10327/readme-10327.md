Client Review Collection & Sentiment Analysis with HighLevel, GPT-4o, Gmail & Slack

https://n8nworkflows.xyz/workflows/client-review-collection---sentiment-analysis-with-highlevel--gpt-4o--gmail---slack-10327


# Client Review Collection & Sentiment Analysis with HighLevel, GPT-4o, Gmail & Slack

### 1. Workflow Overview

This workflow automates the collection and analysis of client reviews following successful deal closures in the HighLevel CRM platform. It targets service-based businesses that want to streamline client feedback gathering, enhance customer engagement, and monitor sentiment in real-time.

The workflow is logically divided into the following blocks:

- **1.1 Input Trigger & Deal Retrieval:** Starts manually or on demand, fetching all won deals from HighLevel.
- **1.2 Deal Validation:** Ensures the fetched deals contain necessary data before proceeding.
- **1.3 AI-Powered Review Request Email Generation:** Uses Azure OpenAI GPT-4o to create personalized, HTML-formatted review request emails.
- **1.4 Email Delivery:** Sends the AI-generated email to clients via Gmail.
- **1.5 Delay & Feedback Retrieval:** Waits 24 hours, then retrieves the client’s reply thread from Gmail.
- **1.6 AI-Based Feedback Summarization:** Uses GPT-4o to summarize client feedback and extract sentiment.
- **1.7 Slack Announcement:** Posts the summarized feedback to a Slack channel for internal visibility.
- **1.8 Error Logging:** Logs any errors encountered during execution to a Google Sheets document for auditing and troubleshooting.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Trigger & Deal Retrieval

**Overview:**  
This block initiates the workflow manually and fetches all deals marked as "Won" from HighLevel CRM.

**Nodes Involved:**  
- When clicking ‘Execute workflow’  
- Fetch All Won Deals from HighLevel

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Entry point for manual execution  
  - *Configuration:* No parameters; triggers workflow on user command  
  - *Inputs:* None  
  - *Outputs:* Connects to "Fetch All Won Deals from HighLevel"  
  - *Edge Cases:* None specific; user must manually trigger

- **Fetch All Won Deals from HighLevel**  
  - *Type:* HighLevel API node  
  - *Role:* Retrieves all deals with status "Won"  
  - *Configuration:*  
    - Resource: Opportunity  
    - Operation: getAll  
    - Filter: status = "won"  
  - *Credentials:* HighLevel OAuth2 account required  
  - *Inputs:* From manual trigger  
  - *Outputs:* Connects to "Validate Deal Fetch Success"  
  - *Edge Cases:* API connection failures, empty result sets, rate limits

---

#### 2.2 Deal Validation

**Overview:**  
Validates that each fetched deal has a non-empty ID to prevent processing incomplete or invalid data.

**Nodes Involved:**  
- Validate Deal Fetch Success

**Node Details:**

- **Validate Deal Fetch Success**  
  - *Type:* If node (conditional check)  
  - *Role:* Filters deals based on presence of deal ID  
  - *Configuration:* Checks if `$json.id` is not empty (strict, case sensitive)  
  - *Inputs:* From "Fetch All Won Deals from HighLevel"  
  - *Outputs:*  
    - True branch: to "Generate Personalized Review Request Email (AI)"  
    - False branch: to "Log Errors in Google Sheets"  
  - *Edge Cases:* Deals with missing or empty IDs; workflow will skip email generation and log error instead

---

#### 2.3 AI-Powered Review Request Email Generation

**Overview:**  
Uses Azure OpenAI GPT-4o to generate a personalized HTML email requesting client feedback and public reviews.

**Nodes Involved:**  
- Configure GPT-4o Model  
- Generate Personalized Review Request Email (AI)

**Node Details:**

- **Configure GPT-4o Model**  
  - *Type:* Langchain Azure OpenAI Chat LM node  
  - *Role:* Sets the GPT-4o model for use in subsequent AI nodes  
  - *Configuration:* Model = "gpt-4o"  
  - *Credentials:* Azure OpenAI API  
  - *Inputs:* None (standalone configuration node)  
  - *Outputs:* Connects as language model input to "Generate Personalized Review Request Email (AI)"  
  - *Edge Cases:* API authentication errors, model availability

- **Generate Personalized Review Request Email (AI)**  
  - *Type:* Langchain Agent node  
  - *Role:* Generates the HTML email body using GPT-4o based on deal data  
  - *Configuration:*  
    - Prompt instructs generation of a professional, grateful, concise email with:  
      - Client name personalization  
      - Thank you message  
      - Two call-to-action buttons (Google Review and internal form links) styled with inline CSS in Techdome blue (#1263ff)  
      - Warm closing line  
    - System message enforces style and output format (HTML only, no markdown)  
  - *Inputs:* Validated deal JSON from "Validate Deal Fetch Success"  
  - *Outputs:* Connects to "Send Review Request Email to Client"  
  - *Edge Cases:* AI generation errors, prompt misinterpretation, incomplete data causing poor personalization

---

#### 2.4 Email Delivery

**Overview:**  
Sends the AI-generated review request email to the client’s email address using Gmail.

**Nodes Involved:**  
- Send Review Request Email to Client

**Node Details:**

- **Send Review Request Email to Client**  
  - *Type:* Gmail node  
  - *Role:* Sends email via Gmail SMTP  
  - *Configuration:*  
    - Recipient: fixed as "newscctv22@gmail.com" (likely placeholder, should be replaced with client email)  
    - Subject: "Thank You for Working with Techdome."  
    - Message body: HTML output from AI node  
  - *Credentials:* Gmail OAuth2 account  
  - *Inputs:* From "Generate Personalized Review Request Email (AI)"  
  - *Outputs:* Connects to "Wait for 24 Hours Before Next Action"  
  - *Edge Cases:* Email sending failures, invalid recipient address, Gmail API quota limits

---

#### 2.5 Delay & Feedback Retrieval

**Overview:**  
Waits 24 hours to allow client responses, then retrieves the full Gmail thread containing client feedback.

**Nodes Involved:**  
- Wait for 24 Hours Before Next Action  
- Retrieve Email Thread for Response

**Node Details:**

- **Wait for 24 Hours Before Next Action**  
  - *Type:* Wait node  
  - *Role:* Pauses workflow execution for 24 hours  
  - *Configuration:* Amount = 24 hours  
  - *Inputs:* From "Send Review Request Email to Client"  
  - *Outputs:* Connects to "Retrieve Email Thread for Response"  
  - *Edge Cases:* Workflow execution time limits, manual interruption

- **Retrieve Email Thread for Response**  
  - *Type:* Gmail node  
  - *Role:* Fetches Gmail thread by thread ID to get client replies  
  - *Configuration:*  
    - Resource: thread  
    - Thread ID: extracted dynamically from previous email send metadata (`$json.threadId`)  
  - *Credentials:* Gmail OAuth2 account  
  - *Inputs:* From "Wait for 24 Hours Before Next Action"  
  - *Outputs:* Connects to "Summarize Client Feedback (AI)"  
  - *Edge Cases:* Invalid or missing thread ID, Gmail API errors

---

#### 2.6 AI-Based Feedback Summarization

**Overview:**  
Uses GPT-4o to create a concise, Slack-formatted summary of the client’s feedback, including sentiment analysis.

**Nodes Involved:**  
- Configure GPT-4o Model1  
- Summarize Client Feedback (AI)

**Node Details:**

- **Configure GPT-4o Model1**  
  - *Type:* Langchain Azure OpenAI Chat LM node  
  - *Role:* Sets GPT-4o model for summarization  
  - *Configuration:* Model = "gpt-4o"  
  - *Credentials:* Azure OpenAI API  
  - *Inputs:* None  
  - *Outputs:* Connects as language model input to "Summarize Client Feedback (AI)"  
  - *Edge Cases:* Same as prior GPT-4o model configuration

- **Summarize Client Feedback (AI)**  
  - *Type:* Langchain Agent node  
  - *Role:* Parses Gmail thread JSON and produces a Slack-ready summary with client name, feedback snippet, and sentiment (positive/neutral/negative)  
  - *Configuration:*  
    - Prompt instructs to produce a concise, human-readable Slack message with markdown formatting  
    - System message focuses on clarity, tone analysis, and client-success context  
  - *Inputs:* Gmail thread JSON from "Retrieve Email Thread for Response"  
  - *Outputs:* Connects to "Announce Review Summary in Slack"  
  - *Edge Cases:* AI misinterpretation, incomplete thread data, ambiguous sentiment

---

#### 2.7 Slack Announcement

**Overview:**  
Posts the summarized client feedback message into a Slack channel to inform the internal team promptly.

**Nodes Involved:**  
- Announce Review Summary in Slack

**Node Details:**

- **Announce Review Summary in Slack**  
  - *Type:* Slack node  
  - *Role:* Sends a message to a Slack user/channel  
  - *Configuration:*  
    - Text: output from AI summarization node  
    - User: fixed Slack user ID (U09HMPVD466)  
  - *Credentials:* Slack API token with chat write permissions  
  - *Inputs:* From "Summarize Client Feedback (AI)"  
  - *Outputs:* None (end of branch)  
  - *Edge Cases:* Slack API errors, invalid user/channel, permissions issues

---

#### 2.8 Error Logging

**Overview:**  
Logs any errors encountered in the workflow to a shared Google Sheets document for audit and troubleshooting.

**Nodes Involved:**  
- Log Errors in Google Sheets

**Node Details:**

- **Log Errors in Google Sheets**  
  - *Type:* Google Sheets node  
  - *Role:* Appends error details to a specified Google Sheet tab named "error log sheet"  
  - *Configuration:*  
    - Operation: append  
    - Document ID and Sheet Name preset (ID: 1Uldk_4BxWbdZTDZxFUeohIfeBmGHHqVEl9Ogb0l6R8Y, Sheet: 1338537721)  
    - Columns: error_id, error (both string)  
  - *Credentials:* Google Sheets OAuth2 account  
  - *Inputs:* From false branch of "Validate Deal Fetch Success"  
  - *Outputs:* None  
  - *Edge Cases:* Google Sheets API limits, authentication errors

---

### 3. Summary Table

| Node Name                         | Node Type                                   | Functional Role                          | Input Node(s)                      | Output Node(s)                                | Sticky Note                                                                                          |
|----------------------------------|---------------------------------------------|----------------------------------------|----------------------------------|-----------------------------------------------|----------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                              | Workflow entry point                    | None                             | Fetch All Won Deals from HighLevel            |                                                                                                |
| Fetch All Won Deals from HighLevel| HighLevel API                              | Retrieve all won deals                  | When clicking ‘Execute workflow’ | Validate Deal Fetch Success                     | ## Deal Fetch & Validation: Fetches all deals with “Won” status from HighLevel and checks validity. |
| Validate Deal Fetch Success       | If Node                                     | Validates deal data presence            | Fetch All Won Deals from HighLevel| Generate Personalized Review Request Email (AI), Log Errors in Google Sheets |                                                                                                |
| Generate Personalized Review Request Email (AI) | Langchain Agent (Azure GPT-4o)             | AI email creation                      | Validate Deal Fetch Success (true branch) | Send Review Request Email to Client            | ## AI-Powered Email Generation: Uses GPT-4o to craft personalized HTML review-request emails.       |
| Configure GPT-4o Model            | Langchain Azure OpenAI Chat LM              | Configure GPT-4o model for email generation | None                             | Generate Personalized Review Request Email (AI) |                                                                                                |
| Send Review Request Email to Client| Gmail Node                                | Send review request email to client    | Generate Personalized Review Request Email (AI) | Wait for 24 Hours Before Next Action           | ## Email Delivery: Sends GPT-generated emails via Gmail to clients; tracks sent message.           |
| Wait for 24 Hours Before Next Action | Wait Node                               | Delay before checking client response  | Send Review Request Email to Client | Retrieve Email Thread for Response              | ## Waiting & Feedback Retrieval: Pauses 24 hours, then fetches Gmail thread for client replies.    |
| Retrieve Email Thread for Response| Gmail Node                                 | Retrieve full Gmail thread by thread ID| Wait for 24 Hours Before Next Action | Summarize Client Feedback (AI)                  |                                                                                                |
| Configure GPT-4o Model1           | Langchain Azure OpenAI Chat LM              | Configure GPT-4o model for feedback summarization | None                             | Summarize Client Feedback (AI)                  |                                                                                                |
| Summarize Client Feedback (AI)   | Langchain Agent (Azure GPT-4o)               | Summarize client feedback and sentiment| Retrieve Email Thread for Response| Announce Review Summary in Slack                | ## AI Feedback Summarization: GPT-4o analyzes client replies and creates Slack-ready summaries.    |
| Announce Review Summary in Slack | Slack Node                                  | Post summarized feedback to Slack      | Summarize Client Feedback (AI)   | None                                          | ## Slack Announcement: Posts summarized client feedback to Slack for internal team visibility.    |
| Log Errors in Google Sheets       | Google Sheets Node                          | Log workflow/API errors                 | Validate Deal Fetch Success (false branch) | None                                          | ## Error Logging: Appends API or workflow errors to Google Sheets for auditing and debugging.      |
| Sticky Note                      | Sticky Note                                 | Workflow overview and setup instructions| None                             | None                                          | See detailed notes in Section 5                                                                    |
| Sticky Note1                     | Sticky Note                                 | Deal fetch & validation explanation    | None                             | None                                          | See detailed notes in Section 5                                                                    |
| Sticky Note2                     | Sticky Note                                 | AI email generation explanation        | None                             | None                                          | See detailed notes in Section 5                                                                    |
| Sticky Note3                     | Sticky Note                                 | Email delivery explanation              | None                             | None                                          | See detailed notes in Section 5                                                                    |
| Sticky Note4                     | Sticky Note                                 | Waiting and feedback retrieval notes    | None                             | None                                          | See detailed notes in Section 5                                                                    |
| Sticky Note5                     | Sticky Note                                 | AI feedback summarization explanation  | None                             | None                                          | See detailed notes in Section 5                                                                    |
| Sticky Note6                     | Sticky Note                                 | Slack announcement explanation          | None                             | None                                          | See detailed notes in Section 5                                                                    |
| Sticky Note7                     | Sticky Note                                 | Error logging explanation                | None                             | None                                          | See detailed notes in Section 5                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**
   - Type: Manual Trigger  
   - Name: "When clicking ‘Execute workflow’"  
   - No parameters required  
   - This starts the workflow manually.

2. **Add HighLevel Node to Fetch Won Deals**
   - Type: HighLevel  
   - Name: "Fetch All Won Deals from HighLevel"  
   - Resource: Opportunity  
   - Operation: getAll  
   - Filters: status = "won"  
   - Credentials: Set up HighLevel OAuth2 API credentials  
   - Connect output of Manual Trigger to this node.

3. **Add If Node to Validate Deal Data**
   - Type: If  
   - Name: "Validate Deal Fetch Success"  
   - Condition: Check if `$json.id` is not empty (strict, case sensitive)  
   - Connect output of "Fetch All Won Deals from HighLevel" to this node.

4. **Add Google Sheets Node for Error Logging**
   - Type: Google Sheets  
   - Name: "Log Errors in Google Sheets"  
   - Operation: Append  
   - Document ID: `1Uldk_4BxWbdZTDZxFUeohIfeBmGHHqVEl9Ogb0l6R8Y`  
   - Sheet Name: `error log sheet` (tab ID: 1338537721)  
   - Columns: error_id (string), error (string)  
   - Credentials: Set Google Sheets OAuth2 account  
   - Connect False output of "Validate Deal Fetch Success" to this node.

5. **Add Azure OpenAI Chat LM Node to Configure GPT-4o (Email Generation)**
   - Type: Langchain Azure OpenAI Chat LM  
   - Name: "Configure GPT-4o Model"  
   - Model: gpt-4o  
   - Credentials: Azure OpenAI API  
   - No input connection needed; standalone config.

6. **Add Langchain Agent Node to Generate Review Email**
   - Type: Langchain Agent  
   - Name: "Generate Personalized Review Request Email (AI)"  
   - Text prompt: Provide client deal JSON and instruct to create an HTML email with:  
     - Personalized client name  
     - Thank you message  
     - Two CTA buttons linking to Google Review and internal form  
     - Warm closing line  
   - System message: Defines tone, style, and output format (HTML only)  
   - Connect True output of "Validate Deal Fetch Success" to this node.  
   - Set "Configure GPT-4o Model" as the language model input.

7. **Add Gmail Node to Send Email**
   - Type: Gmail  
   - Name: "Send Review Request Email to Client"  
   - Recipient: Replace placeholder with client's email from deal data (currently fixed as "newscctv22@gmail.com")  
   - Subject: "Thank You for Working with Techdome."  
   - Message: Use HTML output from AI node  
   - Credentials: Gmail OAuth2 account  
   - Connect output of AI email generation node here.

8. **Add Wait Node for 24 Hours Delay**
   - Type: Wait  
   - Name: "Wait for 24 Hours Before Next Action"  
   - Amount: 24 hours  
   - Connect output of Gmail send node.

9. **Add Gmail Node to Retrieve Email Thread**
   - Type: Gmail  
   - Name: "Retrieve Email Thread for Response"  
   - Resource: thread  
   - Operation: get  
   - Thread ID: dynamically from previous email send metadata (e.g., `$json.threadId`)  
   - Credentials: Same Gmail OAuth2 account  
   - Connect output of Wait node.

10. **Add Second Azure OpenAI Chat LM Node for GPT-4o (Feedback Summarization)**
    - Type: Langchain Azure OpenAI Chat LM  
    - Name: "Configure GPT-4o Model1"  
    - Model: gpt-4o  
    - Credentials: Azure OpenAI API  
    - Standalone node.

11. **Add Langchain Agent Node to Summarize Feedback**
    - Type: Langchain Agent  
    - Name: "Summarize Client Feedback (AI)"  
    - Text prompt: Provide Gmail thread JSON; instruct to create a concise Slack message with client name, feedback snippet, sentiment (positive/neutral/negative)  
    - System message: Focus on clarity and client-success context  
    - Connect output of "Retrieve Email Thread for Response" to this node.  
    - Use "Configure GPT-4o Model1" as language model input.

12. **Add Slack Node to Post Summary**
    - Type: Slack  
    - Name: "Announce Review Summary in Slack"  
    - Text: Output from AI summarization node  
    - User: Set Slack user ID or channel ID for message posting  
    - Credentials: Slack API token with chat write permissions  
    - Connect output of AI summarization node.

13. **Connect Error Logging Node to False Branch of Validation**
    - Already done in step 4.

14. **Test Workflow**
    - Run manually with a dummy “Won” deal to verify end-to-end operation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                            |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------|
| This workflow automates the client-review process after a deal is marked as “Won” in HighLevel CRM. It retrieves won deals, validates them, generates personalized review request emails using Azure OpenAI GPT-4o, sends emails via Gmail, waits 24 hours, fetches client replies, summarizes feedback sentiment via GPT-4o, and posts results in Slack. Errors are logged in Google Sheets. | Workflow purpose and overview explained in the main Sticky Note.                                                           |
| Setup requires connecting the following credentials: HighLevel API, Azure OpenAI (GPT-4o), Gmail, Slack, Google Sheets. Update Sheet IDs, Slack user/channel references, email subject, sender, and review links as needed. Optionally, schedule triggers instead of manual runs.                                                                                                                                | Setup instructions from main Sticky Note.                                                                                   |
| Deal fetch & validation step prevents processing empty or incomplete deals to avoid unnecessary AI calls or email sends.                                                                                                                                                                                                                                                                                         | Sticky Note1 near deal fetch nodes.                                                                                         |
| AI-generated emails are professional, concise, and include brand-styled call-to-action buttons linking to Google Reviews and internal feedback forms.                                                                                                                                                                                                                                                           | Sticky Note2 near AI email generation nodes.                                                                                |
| Email delivery is done via Gmail, with tracking for later retrieval of client responses.                                                                                                                                                                                                                                                                                                                        | Sticky Note3 near Gmail send node.                                                                                          |
| Workflow waits 24 hours post-email before retrieving client feedback to ensure responses are received.                                                                                                                                                                                                                                                                                                           | Sticky Note4 near wait node and Gmail thread retrieval.                                                                     |
| GPT-4o summarizes client replies into concise Slack-ready messages, capturing sentiment and key feedback points.                                                                                                                                                                                                                                                                                                | Sticky Note5 near AI summarization node.                                                                                    |
| Summarized feedback is posted into Slack to notify relevant team members immediately, supporting client success and reputation monitoring.                                                                                                                                                                                                                                                                       | Sticky Note6 near Slack announcement node.                                                                                  |
| All API or workflow errors are appended to a Google Sheet for transparency and troubleshooting.                                                                                                                                                                                                                                                                                                                | Sticky Note7 near Google Sheets error logging node.                                                                         |
| Recommended to replace placeholder emails and Slack user/channel IDs with production values before deployment.                                                                                                                                                                                                                                                                                                  | Important operational note.                                                                                                |
| Google Review Link used in email: https://share.google/W3i4ulISiqRONADvO  
Internal Feedback Form Link: https://docs.google.com/forms/d/e/1FAIpQLScot2dbxFrx7jS8oTbQa3HZy9SSUtuB0iYch5U83UNNEwB9_g/viewform?usp=publish-editor                                                                                                                                                          | Links embedded in AI prompt for email generation.                                                                           |

---

**Disclaimer:**  
The provided text is exclusively derived from an n8n automated workflow. It adheres strictly to all applicable content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly accessible.