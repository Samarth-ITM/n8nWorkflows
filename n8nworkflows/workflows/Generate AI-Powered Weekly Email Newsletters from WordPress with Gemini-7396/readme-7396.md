Generate AI-Powered Weekly Email Newsletters from WordPress with Gemini

https://n8nworkflows.xyz/workflows/generate-ai-powered-weekly-email-newsletters-from-wordpress-with-gemini-7396


# Generate AI-Powered Weekly Email Newsletters from WordPress with Gemini

### 1. Workflow Overview

This n8n workflow automates the generation and distribution of a weekly email newsletter based on recent WordPress blog posts, leveraging AI to create engaging content. It is designed to run every Friday at 10 AM, fetching the latest posts, creating a formatted newsletter via Google Gemini AI, and sending it to a predefined subscriber list via SMTP email.

The workflow is logically divided into these blocks:

- **1.1 Schedule Trigger & Content Fetching:** Initiates the workflow weekly and retrieves recent WordPress posts.
- **1.2 Post Existence Check:** Determines if new posts are present to process.
- **1.3 AI Newsletter Generation:** Uses Google Gemini AI to create a personalized newsletter from the posts.
- **1.4 Content Parsing:** Cleans and structures the AI output into usable email content.
- **1.5 Email Sending:** Dispatches the formatted newsletter to subscribers via SMTP.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule Trigger & Content Fetching

- **Overview:**  
  This block triggers the workflow every Friday at 10 AM and fetches all recent posts from the WordPress site using configured credentials.

- **Nodes Involved:**  
  - Weekly Friday 10AM (Schedule Trigger)  
  - Fetch Recent Posts (WordPress)

- **Node Details:**

  **Weekly Friday 10AM**  
  - Type: Schedule Trigger  
  - Role: Starts the workflow on a cron schedule  
  - Configuration: Cron expression `0 10 * * 5` (10:00 AM every Friday)  
  - Inputs: None (trigger node)  
  - Outputs: Connects to Fetch Recent Posts  
  - Version: 1.2  
  - Edge Cases: Ensure server time zone matches expected schedule; cron syntax errors may halt triggering.

  **Fetch Recent Posts**  
  - Type: WordPress node  
  - Role: Retrieves all recent posts from WordPress using API credentials  
  - Configuration: Operation set to `getAll` without additional filters, fetching all posts  
  - Inputs: Trigger from schedule node  
  - Outputs: Passes posts to Check Posts Exist node  
  - Credentials: Requires WordPress API credentials configured in n8n  
  - Version: 1  
  - Edge Cases: API authentication failures, network timeouts, empty post lists, rate limiting.

#### 2.2 Post Existence Check

- **Overview:**  
  Checks if any posts were fetched to decide whether to proceed or stop workflow execution.

- **Nodes Involved:**  
  - Check Posts Exist (If node)

- **Node Details:**

  **Check Posts Exist**  
  - Type: If node  
  - Role: Conditional branching based on posts count  
  - Configuration: Condition checks if number of posts (`{{$json.length}}`) is greater than 0  
  - Inputs: From Fetch Recent Posts  
  - Outputs: True branch leads to AI Newsletter Creator; false branch terminates workflow  
  - Version: 2  
  - Edge Cases: Failure if `$json.length` is undefined or if no posts are returned; should handle empty arrays gracefully.

#### 2.3 AI Newsletter Generation

- **Overview:**  
  Uses the Google Gemini AI model (via LangChain agent) to generate a weekly newsletter draft based on the retrieved posts, including subject, greeting, summaries, and call-to-action.

- **Nodes Involved:**  
  - AI Newsletter Creator (LangChain Agent)  
  - Google Gemini Chat Model (Language Model node)

- **Node Details:**

  **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini Chat Model  
  - Role: AI language model backend for newsletter creation  
  - Configuration: Default options, credentials linked to Google Palm API (Gemini)  
  - Inputs: None directly; connected as AI language model to AI Newsletter Creator  
  - Version: 1  
  - Credentials: Google Palm API key (Google Gemini)  
  - Edge Cases: API rate limits, authentication errors, latency, or model unavailability.

  **AI Newsletter Creator**  
  - Type: LangChain Agent node  
  - Role: Sends posts data and system prompt to Google Gemini to generate newsletter content  
  - Configuration: System message instructs to craft an engaging email newsletter in HTML format including subject, greeting, intro, summaries, CTA, and sign-off in conversational style  
  - Inputs: Receives posts from Check Posts Exist (true branch) and AI language model from Google Gemini Chat Model  
  - Outputs: Passes AI-generated content to Parse Newsletter Content node  
  - Version: 2.2  
  - Edge Cases: AI output format may vary; failure if AI response is malformed or API errors occur.

#### 2.4 Content Parsing

- **Overview:**  
  Processes the AI output to remove markdown code blocks, parse JSON if present, and standardize the newsletter subject and content for sending.

- **Nodes Involved:**  
  - Parse Newsletter Content (Code node)

- **Node Details:**

  **Parse Newsletter Content**  
  - Type: Code (JavaScript) node  
  - Role: Cleans AI output, strips markdown syntax, attempts to parse JSON, and creates structured email content  
  - Configuration: JS code removes ```json and ``` blocks, parses JSON or defaults to plain text with fallback subject including current date  
  - Inputs: AI-generated newsletter content from AI Newsletter Creator  
  - Outputs: Structured JSON with `subject`, `content`, and current ISO date to Send Newsletter node  
  - Version: 2  
  - Edge Cases: Parsing errors if AI output is not valid JSON or missing expected fields; fallback ensures no failure but may produce less structured emails.

#### 2.5 Email Sending

- **Overview:**  
  Sends the finalized newsletter via SMTP to a predefined list of subscribers with HTML formatting.

- **Nodes Involved:**  
  - Send Newsletter (Email Send node)

- **Node Details:**

  **Send Newsletter**  
  - Type: Email Send  
  - Role: Sends the newsletter email using SMTP credentials  
  - Configuration:  
    - Subject: Dynamic from parsed content  
    - HTML Body: Dynamic newsletter content  
    - Recipients: Hardcoded list of subscriber emails (e.g., subscribers@yourlist.com, subscriber2@email.com, subscriber3@email.com)  
    - From Email: newsletter@yoursite.com  
    - Email format: HTML  
  - Inputs: From Parse Newsletter Content  
  - Credentials: SMTP credentials configured in n8n (example given is SMTP account with ID)  
  - Version: 2  
  - Edge Cases: SMTP authentication failures, invalid email addresses, email sending limits, HTML rendering issues in recipients' clients.

---

### 3. Summary Table

| Node Name             | Node Type                        | Functional Role                      | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                           |
|-----------------------|---------------------------------|------------------------------------|------------------------|-------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Sticky Note           | Sticky Note                     | Documentation / Notes               | None                   | None                    | **Weekly Email Newsletter Generator**<br>- Fetches latest WordPress posts every Friday<br>- AI creates engaging newsletter content<br>- Sends formatted emails to subscriber list<br>- Tracks included posts to avoid duplicates<br><br>**Setup Required:**<br>1. Configure WordPress credentials<br>2. Set up SMTP email credentials<br>3. Replace emails in Send Newsletter node<br>4. Set Google Gemini API credentials<br><br>**Customization:**<br>- Change schedule<br>- Adjust posts count<br>- Modify email template<br>- Add intro/outro<br><br>**Need Help?**<br>Contact: mailto:david@daexai.com |
| Weekly Friday 10AM    | Schedule Trigger                 | Triggers workflow weekly on Friday | None                   | Fetch Recent Posts       | See Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Fetch Recent Posts    | WordPress                       | Fetches recent WordPress posts     | Weekly Friday 10AM      | Check Posts Exist        | See Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Check Posts Exist     | If                             | Checks if posts exist to continue  | Fetch Recent Posts      | AI Newsletter Creator    | See Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Google Gemini Chat Model | LangChain Google Gemini Chat Model | Provides AI language model backend | None (linked as AI LM) | AI Newsletter Creator (as AI LM) | See Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                        |
| AI Newsletter Creator | LangChain Agent                 | Generates newsletter content via AI | Check Posts Exist (true) & Google Gemini Chat Model | Parse Newsletter Content | See Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Parse Newsletter Content | Code (JavaScript)               | Parses AI output for email sending | AI Newsletter Creator  | Send Newsletter          | See Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                        |
| Send Newsletter       | Email Send                     | Sends newsletter email via SMTP   | Parse Newsletter Content | None                    | See Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Name: `Weekly Friday 10AM`  
   - Type: Schedule Trigger  
   - Parameters: Set Cron expression to `0 10 * * 5` (10 AM every Friday)  
   - No credentials needed  
   - This node will start the workflow weekly.

2. **Create WordPress Node**  
   - Name: `Fetch Recent Posts`  
   - Type: WordPress  
   - Credentials: Configure WordPress API credentials in n8n  
   - Operation: `getAll` (fetch all posts)  
   - Connect input from `Weekly Friday 10AM` output.

3. **Create If Node**  
   - Name: `Check Posts Exist`  
   - Type: If  
   - Condition: Check if `{{$json.length}} > 0` (number of posts fetched)  
   - Connect input from `Fetch Recent Posts` output.  
   - True branch to AI nodes; false branch leaves workflow (no further connection).

4. **Create LangChain Google Gemini Chat Model Node**  
   - Name: `Google Gemini Chat Model`  
   - Type: LangChain Google Gemini Chat Model  
   - Credentials: Configure Google Palm API credentials (Google Gemini) in n8n  
   - No input connections.

5. **Create LangChain Agent Node**  
   - Name: `AI Newsletter Creator`  
   - Type: LangChain Agent  
   - Parameters:  
     - System Message: "You are an email newsletter writer. Create an engaging weekly newsletter based on the provided blog posts. Include: 1) Catchy subject line, 2) Personal greeting, 3) Brief intro about this week's content, 4) Formatted summaries of each post with compelling descriptions, 5) Call-to-action to read full posts, 6) Friendly sign-off. Keep it conversational and engaging. Format in HTML for email."  
   - Connect input from `Check Posts Exist` true branch.  
   - Under AI Language Model option, select `Google Gemini Chat Model`.

6. **Create Code Node**  
   - Name: `Parse Newsletter Content`  
   - Type: Code (JavaScript)  
   - Code:  
     ```javascript
     const items = $input.all();

     return items.map(item => {
       let aiOutput = item.json.output;
       aiOutput = aiOutput.replace(/```json\s*/, '').replace(/```\s*$/, '');
       let parsedOutput;
       try {
         parsedOutput = JSON.parse(aiOutput.trim());
       } catch (e) {
         parsedOutput = {
           subject: "Weekly Newsletter - " + new Date().toLocaleDateString(),
           content: aiOutput
         };
       }
       return {
         json: {
           subject: parsedOutput.subject || "Weekly Newsletter - " + new Date().toLocaleDateString(),
           content: parsedOutput.content || aiOutput,
           date: new Date().toISOString()
         }
       };
     });
     ```  
   - Connect input from `AI Newsletter Creator`.

7. **Create Email Send Node**  
   - Name: `Send Newsletter`  
   - Type: Email Send  
   - Credentials: Configure SMTP credentials (Gmail, SendGrid, Mailgun, etc.) in n8n  
   - Parameters:  
     - To Email: `subscribers@yourlist.com,subscriber2@email.com,subscriber3@email.com` (replace with your subscriber emails)  
     - From Email: `newsletter@yoursite.com` (replace with your sending email)  
     - Subject: Expression `{{$json.subject}}`  
     - HTML Body: Expression `{{$json.content}}`  
     - Email Format: HTML  
   - Connect input from `Parse Newsletter Content`.

8. **Add a Sticky Note (optional)**  
   - Add a Sticky Note to document the workflow purpose, setup, customization tips, and support contact.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                  | Context or Link                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------|
| Workflow fetches latest WordPress posts every Friday, uses Google Gemini AI to create engaging email newsletters, and sends them via SMTP to your subscriber list. Setup requires WordPress, SMTP, and Google Gemini credentials.               | Workflow overview and setup         |
| To customize: adjust schedule, number of posts fetched, email recipient list, email template, and AI prompt to tune newsletter tone and content.                                                                                             | Customization guidance              |
| For personalized n8n coaching or one-on-one consultation, contact: mailto:david@daexai.com                                                                                                                                                    | Support contact                    |
| Google Gemini AI is accessed via Google Palm API credentials; ensure valid API key with sufficient quota.                                                                                                                                      | Google Gemini API setup             |
| SMTP credentials must be valid and capable of sending to your subscriber domain; watch for email sending limits and spam filters.                                                                                                            | SMTP email sending best practices  |

---

**Disclaimer:** The text provided is exclusively from an automated workflow created with n8n, respecting all current content policies without illegal, offensive, or protected elements. All data processed is legal and public.