Jobs Newsletter Automation System (N8N, Bolt.new, RapidAPI, Mails.so & ChatGPT)

https://n8nworkflows.xyz/workflows/jobs-newsletter-automation-system--n8n--bolt-new--rapidapi--mails-so---chatgpt--4470


# Jobs Newsletter Automation System (N8N, Bolt.new, RapidAPI, Mails.so & ChatGPT)

---

### 1. Workflow Overview

This workflow, titled **"Jobs Newsletter Automation System (N8N, Bolt.new, RapidAPI, Mails.so & ChatGPT)"**, automates the management of a job opportunities newsletter subscription service. The system handles new subscriptions, email validation, unsubscriptions, job scraping, newsletter generation, and sending personalized emails to subscribers. It integrates multiple services including Google Sheets for subscriber storage, RapidAPI for job data, Mails.so for email validation, OpenAI’s GPT-3.5 for job description summarization, and SMTP/Gmail for email delivery.

The workflow is logically divided into three main blocks:

- **1.1 Subscription Management:** Handles incoming subscription requests via webhook, validates emails, checks for duplicates, and manages subscriber data in Google Sheets, sending appropriate welcome or notification emails.

- **1.2 Unsubscription Management:** Processes unsubscription requests, updates subscriber status in sheets, removes subscribers from active lists, and sends confirmation emails.

- **1.3 Newsletter Generation and Sending:** Periodically triggered to scrape recent job postings, summarize descriptions, format a personalized HTML newsletter, and send it to all active subscribers.

---

### 2. Block-by-Block Analysis

#### 1.1 Subscription Management

**Overview:**  
This block processes incoming subscription requests, validates the email address using an external API, checks if the email already exists in the database, and updates subscriber records accordingly. It sends either a welcome email or an "already subscribed" notification.

**Nodes Involved:**  
- Webhook  
- Confirm Email Validity  
- If1  
- Get All Subscribers  
- If email is already in database  
- Send Welcome Email  
- Send Email1 (Already Subscribed Notification)  
- Add Email to All Subscribers Sheet  
- Add Email to Subscribed Sheet  
- Notification of Invalid Email  
- Notification of Successful Subscription  

**Node Details:**

- **Webhook**  
  - *Type:* Webhook (HTTP POST)  
  - *Role:* Entry point to receive subscription requests with subscriber details (email, name).  
  - *Configuration:* Exposes a POST endpoint with path `5ec8dcf6-b2ba-4fb3-bf62-d253d2f39f02`.  
  - *Input:* HTTP POST with JSON body containing at least `email` and `name`.  
  - *Output:* Passes request data downstream.  
  - *Edge Cases:* Request missing email/name, invalid HTTP method.

- **Confirm Email Validity**  
  - *Type:* HTTP Request  
  - *Role:* Validates the provided email using Mails.so API.  
  - *Configuration:* GET request to `https://api.mails.so/v1/validate?email={{ email }}` with header `x-mails-api-key` (user must provide).  
  - *Input:* Email from Webhook node.  
  - *Output:* JSON with email validation results (e.g., deliverability, MX record presence).  
  - *Edge Cases:* Invalid API key, network timeout, API response errors.

- **If1**  
  - *Type:* If condition  
  - *Role:* Checks if email is valid (not undeliverable or has MX record).  
  - *Configuration:* Passes if email is not undeliverable OR MX record is not empty.  
  - *Input:* Email validation response.  
  - *Output:* Routes valid and invalid email paths.  
  - *Edge Cases:* Unexpected API response structure.

- **Get All Subscribers**  
  - *Type:* Google Sheets (read)  
  - *Role:* Checks if the email already exists in the "All Subscribers" sheet.  
  - *Configuration:* Filters rows by email.  
  - *Input:* Email from Webhook.  
  - *Output:* Matching subscriber rows or empty.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Edge Cases:* API quota limit, sheet access errors.

- **If email is already in database**  
  - *Type:* If condition  
  - *Role:* Determines if the subscriber already exists by checking presence of `row_number` in sheet data.  
  - *Output:* Branches to existing email or new subscription flow.

- **Send Welcome Email**  
  - *Type:* Email Send (SMTP)  
  - *Role:* Sends a personalized welcome email to new subscribers.  
  - *Configuration:* HTML email with branding, personalized with subscriber’s name and email.  
  - *Credentials:* SMTP account.  
  - *Edge Cases:* SMTP authentication failure, email rejection.

- **Send Email1 (Already Subscribed Notification)**  
  - *Type:* Email Send (SMTP)  
  - *Role:* Notifies users attempting to resubscribe that they are already subscribed.  
  - *Configuration:* HTML email informing existing subscription status.  
  - *Credentials:* SMTP account.  
  - *Edge Cases:* Same as above.

- **Add Email to All Subscribers Sheet**  
  - *Type:* Google Sheets (append)  
  - *Role:* Adds new subscriber info to the "All Subscribers" sheet with subscription status and date.  
  - *Input:* Subscriber name, email, current date, status "Subscribed".  
  - *Credentials:* Google Sheets OAuth2.  
  - *Edge Cases:* Write permission errors, quota limits.

- **Add Email to Subscribed Sheet**  
  - *Type:* Google Sheets (append or update)  
  - *Role:* Ensures subscriber is present and updated in the "Subscribed" sheet.  
  - *Input:* Subscriber name and email.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Edge Cases:* Same as above.

- **Notification of Invalid Email**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends back an HTTP response indicating email validation failure.  
  - *Content:* Text message with subscriber name and invalid email.  
  - *Edge Cases:* None specific.

- **Notification of Successful Subscription**  
  - *Type:* Respond to Webhook  
  - *Role:* Sends back confirmation that subscription succeeded and welcome email sent.  
  - *Content:* Text message with subscriber name confirmation.

---

#### 1.2 Unsubscription Management

**Overview:**  
Manages unsubscription requests by updating subscriber status in Google Sheets, moving subscriber data to the "Unsubscribed" sheet, deleting from active subscribers, and sending a confirmation email.

**Nodes Involved:**  
- Webhook1  
- get rows in all subscribers sheet  
- change status to unsubscribed  
- add to unsubscribed sheet  
- get row in subscribed sheet  
- delete row from subscribed  
- Send Email (Unsubscription Confirmation)  

**Node Details:**

- **Webhook1**  
  - *Type:* Webhook (HTTP POST)  
  - *Role:* Entry point for unsubscription requests with email data.  
  - *Configuration:* POST endpoint with path `709bb303-2c56-456a-b039-db5254c14328`.  
  - *Input:* Email to unsubscribe.  
  - *Output:* Passes email data downstream.

- **get rows in all subscribers sheet**  
  - *Type:* Google Sheets (read)  
  - *Role:* Finds subscriber’s row in "All Subscribers" sheet by email.  
  - *Input:* Email from webhook.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Output:* Subscriber row data.

- **change status to unsubscribed**  
  - *Type:* Google Sheets (update)  
  - *Role:* Updates subscriber's "Subscription Status" column to "Unsubscribed" in "All Subscribers".  
  - *Input:* Subscriber email and row info.  
  - *Credentials:* Google Sheets OAuth2.

- **add to unsubscribed sheet**  
  - *Type:* Google Sheets (append)  
  - *Role:* Adds subscriber info to "Unsubscribed" sheet.  
  - *Input:* Name and email from previous node.  
  - *Credentials:* Google Sheets OAuth2.

- **get row in subscribed sheet**  
  - *Type:* Google Sheets (read)  
  - *Role:* Retrieves subscriber row in "Subscribed" sheet by email.  
  - *Input:* Email from unsubscribed sheet append node.  
  - *Credentials:* Google Sheets OAuth2.

- **delete row from subscribed**  
  - *Type:* Google Sheets (delete)  
  - *Role:* Removes subscriber row from "Subscribed" sheet.  
  - *Input:* Row number from previous node.  
  - *Credentials:* Google Sheets OAuth2.

- **Send Email (Unsubscription Confirmation)**  
  - *Type:* Email Send (SMTP)  
  - *Role:* Sends an email confirming successful unsubscription.  
  - *Configuration:* HTML email with branding and instructions.  
  - *Credentials:* SMTP account.  
  - *Edge Cases:* SMTP errors, invalid email address.

---

#### 1.3 Newsletter Generation and Sending

**Overview:**  
Triggered on schedule, this block retrieves all subscribed users, scrapes recent job listings from multiple sources via RapidAPI, summarizes job descriptions using OpenAI GPT-3.5, formats a comprehensive HTML newsletter, and sends it to each subscriber.

**Nodes Involved:**  
- Schedule Trigger  
- Get Subscribers  
- If there are subscribers  
- Get Jobs  
- Split Out  
- Summarize Job Descriptions  
- Aggregate  
- Aggregate1  
- Merge1  
- Format Newsletter  
- Send Newsletter  
- Stop and Error  

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Scheduler  
  - *Role:* Starts the newsletter generation process on a recurring interval (default: every 1 minute/hour/day depending on config).  
  - *Output:* Triggers downstream nodes.

- **Get Subscribers**  
  - *Type:* Google Sheets (read)  
  - *Role:* Retrieves all active subscribers from the "Subscribed" sheet.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Output:* List of subscriber emails and names.

- **If there are subscribers**  
  - *Type:* If condition  
  - *Role:* Checks if there is at least one subscriber to send newsletter to.  
  - *Output:* Routes to job scraping or stops workflow.

- **Get Jobs**  
  - *Type:* HTTP Request  
  - *Role:* Fetches recent full-time remote product manager jobs from various sites via RapidAPI.  
  - *Configuration:* POST with JSON specifying search parameters, includes RapidAPI key and host headers.  
  - *Output:* JSON list of raw job postings.  
  - *Edge Cases:* API rate limits, invalid API key, network errors.

- **Split Out**  
  - *Type:* Split Out  
  - *Role:* Splits the list of jobs into individual job items for processing.  
  - *Output:* Single job per execution.

- **Summarize Job Descriptions**  
  - *Type:* OpenAI (GPT-3.5-Turbo)  
  - *Role:* Summarizes each job description into one sentence using AI.  
  - *Input:* Job description text.  
  - *Credentials:* OpenAI API key.  
  - *Edge Cases:* API errors, rate limits.

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Collects summarized job descriptions back into a list.  
  - *Output:* Aggregated summaries.

- **Aggregate1**  
  - *Type:* Aggregate  
  - *Role:* Collects original job data into a list for newsletter content.  
  - *Output:* Aggregated job data.

- **Merge1**  
  - *Type:* Merge  
  - *Role:* Combines aggregated job data and summaries into a unified dataset for newsletter formatting.  
  - *Output:* Combined job info with summaries.

- **Format Newsletter**  
  - *Type:* Code (JavaScript)  
  - *Role:* Generates a styled HTML newsletter embedding job info, summaries, subscriber name, current date, and unsubscribe links.  
  - *Input:* Aggregated and merged job data and subscriber info.  
  - *Output:* HTML string for newsletter email.

- **Send Newsletter**  
  - *Type:* Email Send (SMTP)  
  - *Role:* Sends the formatted newsletter HTML email to each subscriber.  
  - *Configuration:* Uses subscriber’s email and personalized content.  
  - *Credentials:* SMTP account.  
  - *Edge Cases:* SMTP errors, invalid recipient emails.

- **Stop and Error**  
  - *Type:* Stop and Error  
  - *Role:* Stops workflow execution if no subscribers are found, with an error message.

---

### 3. Summary Table

| Node Name                      | Node Type                 | Functional Role                                | Input Node(s)                     | Output Node(s)                            | Sticky Note                                  |
|--------------------------------|---------------------------|-----------------------------------------------|----------------------------------|------------------------------------------|----------------------------------------------|
| Webhook                        | Webhook                   | Entry point for subscription requests         | -                                | Confirm Email Validity, Get All Subscribers| Manage Subscriptions with bolt.new frontend |
| Confirm Email Validity         | HTTP Request              | Validate subscriber email via Mails.so API    | Webhook                          | If1                                       |                                              |
| If1                           | If                        | Check email validity                           | Confirm Email Validity            | Send Welcome Email (valid), Notification of Invalid Email (invalid) |                                              |
| Get All Subscribers            | Google Sheets (read)      | Check if email exists in "All Subscribers"    | Webhook                         | If email is already in database            |                                              |
| If email is already in database| If                        | Branch if email exists in database             | Get All Subscribers              | Send Email1 (exists), Confirm Email Validity (not exists)|                                              |
| Send Welcome Email             | Email Send (SMTP)         | Send welcome email to new subscribers          | If1 (valid)                      | Merge                                     |                                              |
| Send Email1                   | Email Send (SMTP)         | Notify user email already subscribed           | If email is already in database  | Notification of Already Exists             |                                              |
| Add Email to All Subscribers Sheet| Google Sheets (append) | Add new subscriber to "All Subscribers" sheet | Send Welcome Email               | Merge                                     |                                              |
| Add Email to Subscribed Sheet | Google Sheets (append/update)| Add/update subscriber in "Subscribed" sheet  | Send Welcome Email               | Merge                                     |                                              |
| Notification of Invalid Email | Respond to Webhook        | Respond with invalid email message             | If1 (invalid)                   | -                                         |                                              |
| Notification of Successful Subscription| Respond to Webhook | Respond with success message                    | Merge                          | -                                         |                                              |
| Webhook1                      | Webhook                   | Entry point for unsubscription requests        | -                                | get rows in all subscribers sheet         | Manage Unsubscriptions                        |
| get rows in all subscribers sheet| Google Sheets (read)    | Lookup subscriber by email in "All Subscribers"| Webhook1                      | change status to unsubscribed              |                                              |
| change status to unsubscribed | Google Sheets (update)    | Update subscription status to "Unsubscribed"  | get rows in all subscribers sheet| add to unsubscribed sheet                  |                                              |
| add to unsubscribed sheet     | Google Sheets (append)    | Append subscriber to "Unsubscribed" sheet      | change status to unsubscribed    | get row in subscribed sheet                |                                              |
| get row in subscribed sheet   | Google Sheets (read)      | Find subscriber row in "Subscribed" sheet      | add to unsubscribed sheet        | delete row from subscribed                  |                                              |
| delete row from subscribed    | Google Sheets (delete)    | Remove subscriber from "Subscribed" sheet      | get row in subscribed sheet      | Send Email                                 |                                              |
| Send Email                   | Email Send (SMTP)         | Send unsubscription confirmation email         | delete row from subscribed       | -                                         |                                              |
| Schedule Trigger             | Schedule Trigger          | Trigger newsletter generation periodically     | -                                | Get Subscribers                            | Scrap Jobs, Format Newsletter and Send to Subscribers |
| Get Subscribers             | Google Sheets (read)      | Get all active subscribers                      | Schedule Trigger                | If there are subscribers                   |                                              |
| If there are subscribers     | If                        | Check if subscribers exist                      | Get Subscribers                 | Get Jobs (yes), Stop and Error (no)       |                                              |
| Get Jobs                    | HTTP Request              | Fetch recent job postings via RapidAPI         | If there are subscribers        | Split Out                                  |                                              |
| Split Out                  | Split Out                 | Split job list into individual jobs            | Get Jobs                       | Summarize Job Descriptions, Aggregate1     |                                              |
| Summarize Job Descriprions | OpenAI (GPT-3.5 Turbo)    | Summarize each job description                  | Split Out                     | Aggregate                                   |                                              |
| Aggregate                  | Aggregate                 | Aggregate summarized job descriptions           | Summarize Job Descriptions     | Merge1                                    |                                              |
| Aggregate1                 | Aggregate                 | Aggregate raw job data                           | Split Out                     | Merge1                                    |                                              |
| Merge1                    | Merge                     | Combine aggregated job data and summaries      | Aggregate, Aggregate1          | Format Newsletter                          |                                              |
| Format Newsletter          | Code (JavaScript)         | Generate HTML newsletter from job data          | Merge1                        | Send Newsletter                            |                                              |
| Send Newsletter           | Email Send (SMTP)         | Send personalized newsletter to subscribers    | Format Newsletter, If there are subscribers| -                                  |                                              |
| Stop and Error            | Stop and Error            | Stop workflow if no subscribers found           | If there are subscribers (no)  | -                                         |                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (Subscription Entry)**
   - Type: Webhook (POST)
   - Path: `5ec8dcf6-b2ba-4fb3-bf62-d253d2f39f02`
   - Purpose: Accept subscription requests with JSON body (`email`, `name`).

2. **Add HTTP Request Node for Email Validation**
   - URL: `https://api.mails.so/v1/validate?email={{ $json.body.email }}`
   - Method: GET
   - Headers: `x-mails-api-key` with your API key.
   - Connect from Webhook.

3. **Add If Node to Check Email Validity**
   - Condition: Email result does not contain "undeliverable" OR MX record is not empty.
   - Connect from HTTP Request.

4. **Add Google Sheets Node to Read "All Subscribers" Sheet**
   - Operation: Read rows filtered by `Email Address` = `{{ $json.body.email }}`
   - Document and sheet IDs set to your Google Sheets document.
   - Connect from Webhook.

5. **Add If Node to Check if Email Exists in Database**
   - Condition: Check if `row_number` exists (meaning email found).
   - Connect from Google Sheets node.

6. **Setup Email Send Node to Send "Already Subscribed" Email**
   - Use SMTP credentials.
   - HTML content: Inform user their email is already subscribed.
   - Connect from "email exists" branch of the If node.

7. **Connect "Not Exists" Branch to Confirm Email Validity Node**

8. **Add Email Send Node to Send Welcome Email**
   - Use SMTP credentials.
   - Personalized HTML content welcoming user.
   - Connect from valid email branch of If1.

9. **Add Google Sheets Append Node to Add Email to "All Subscribers"**
   - Append name, email, subscription date (current date), and "Subscribed" status.
   - Connect from Send Welcome Email.

10. **Add Google Sheets Append or Update Node to Add Email to "Subscribed" Sheet**
    - Append or update subscriber entry with name and email.
    - Connect from Send Welcome Email.

11. **Add Merge Node to aggregate welcome email and sheet updates outputs**
    - Connect from Send Welcome Email, Add Email to All Subscribers, Add Email to Subscribed Sheet.
    - Connect to Respond to Webhook node for success message.

12. **Add Respond to Webhook Node for Invalid Email Notification**
    - Connect from invalid email branch of If1.
    - Text response explaining invalid email.

13. **Add Respond to Webhook Node for Successful Subscription**
    - Connect from Merge node.
    - Text response confirming subscription and welcome email sent.

---

14. **Create Webhook Node (Unsubscription Entry)**
    - Type: Webhook (POST)
    - Path: `709bb303-2c56-456a-b039-db5254c14328`
    - Purpose: Accept unsubscription requests with JSON body containing email.

15. **Add Google Sheets Read Node to Find Subscriber in "All Subscribers" Sheet**
    - Filter by `Email Address` = `{{ $json.body.email }}`
    - Connect from Webhook1.

16. **Add Google Sheets Update Node to Change Status to "Unsubscribed"**
    - Update `Subscription Status` column to "Unsubscribed" for matched row.
    - Connect from previous node.

17. **Add Google Sheets Append Node to Add Subscriber to "Unsubscribed" Sheet**
    - Append name and email.
    - Connect from update node.

18. **Add Google Sheets Read Node to Get Subscriber Row in "Subscribed" Sheet**
    - Filter by email.
    - Connect from append node.

19. **Add Google Sheets Delete Node to Remove Subscriber from "Subscribed" Sheet**
    - Delete row using row number from previous node.
    - Connect from read node.

20. **Add Email Send Node to Send Unsubscription Confirmation Email**
    - Use SMTP credentials.
    - HTML content confirming unsubscription.
    - Connect from delete node.

---

21. **Create Schedule Trigger Node**
    - Set desired interval (e.g., daily at specific time).

22. **Add Google Sheets Read Node to Get All Active Subscribers from "Subscribed" Sheet**

23. **Add If Node to Check if Subscribers Exist**
    - Condition: Check if any email addresses returned.

24. **Add HTTP Request Node to Call RapidAPI Jobs Search**
    - POST with JSON body specifying search for product manager remote jobs.
    - Include RapidAPI key and host headers.
    - Connect from "subscribers exist" branch.

25. **Add Split Out Node to Split Job Listings**

26. **Add OpenAI Node to Summarize Job Descriptions**
    - Model: GPT-3.5-Turbo
    - Prompt: Summarize job description to one sentence.
    - Connect from Split Out.

27. **Add Aggregate Node to Collect Summarized Descriptions**

28. **Add Aggregate Node (Aggregate1) to Collect Raw Job Data**

29. **Add Merge Node to Combine Summaries and Raw Job Data**

30. **Add Code Node to Format HTML Newsletter**
    - Use provided JavaScript code to generate styled HTML.
    - Connect from Merge.

31. **Add Email Send Node to Send Newsletter to Each Subscriber**
    - Use SMTP credentials.
    - Personalized HTML newsletter.
    - Connect from Code node and If node for subscribers.

32. **Add Stop and Error Node**
    - Connect from "no subscribers" branch in If node.
    - Error message: "No Subscribers Found".

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                                                                         |
|------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| Manage Subscriptions with bolt.new frontend                                 | Sticky note near Webhook and subscription management nodes, suggesting frontend integration.          |
| Scrap Jobs, Format Newsletter and Send to Subscribers                       | Sticky note near Schedule Trigger and newsletter sending nodes, describing their purpose.             |
| Manage Unsubscriptions                                                      | Sticky note near unsubscription nodes.                                                                |
| Gmail Send Email Alternative (Newsletter)                                  | Sticky note near disabled Gmail node alternative for sending newsletters.                              |
| Gmail Send Email Alternative (Unsubscription)                              | Sticky note near disabled Gmail node alternative for unsubscription confirmation emails.              |
| Gmail Send Email Alternative (Email Exists)                               | Sticky note near disabled Gmail node for "email already exists" notifications.                         |
| Gmail Send Email Alternative (Welcome Email)                              | Sticky note near disabled Gmail node for welcome emails.                                              |
| Terms & Conditions and Privacy Policy links embedded in email footers      | https://magnificent-tanuki-11356e.netlify.app/terms and https://magnificent-tanuki-11356e.netlify.app/privacy |
| Job Opportunities Newsletter branding and footer links                     | https://magnificent-tanuki-11356e.netlify.app/                                                        |

---

This documentation provides a complete structural and functional understanding of the "Jobs Newsletter Automation System" workflow, enabling reproduction, modification, and troubleshooting by advanced users or AI agents.