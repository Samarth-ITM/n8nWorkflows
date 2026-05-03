Automated B2B Lead Generation & Cold Emails with OpenAI, Apify, Gmail & Telegram

https://n8nworkflows.xyz/workflows/automated-b2b-lead-generation---cold-emails-with-openai--apify--gmail---telegram-9816


# Automated B2B Lead Generation & Cold Emails with OpenAI, Apify, Gmail & Telegram

### 1. Workflow Overview

This workflow automates B2B lead generation and cold email outreach, integrating AI-powered email content creation, business data scraping, and communication tracking via Google Sheets and Telegram. It is designed for lead generation agencies or sales teams targeting businesses by type and location, automating the end-to-end process from lead collection to sending customized cold emails and notifying users.

Logical blocks include:

- **1.1 Input Reception:** Collect user input via an interactive form describing the target business type, location, number of leads, and email style.
- **1.2 Business Data Acquisition:** Query Apify API to search for businesses matching input criteria.
- **1.3 Data Filtering:** Filter businesses to retain only those with websites.
- **1.4 Email Address Extraction:** Use OpenAI to extract valid email addresses from business websites.
- **1.5 Data Recording:** Store business contact info and status in Google Sheets.
- **1.6 Email Generation and Sending Loop:** Loop through filtered businesses, generate cold email content with OpenAI, send emails via Gmail, update Google Sheets, and notify via Telegram.
- **1.7 Execution Control and Delays:** Implement waiting and control mechanisms to pace email dispatch.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

**Overview:**  
This block collects lead generation parameters from the user through a web form.

**Nodes Involved:**  
- On form submission  
- Sticky Note1  

**Node Details:**  

- **On form submission**  
  - *Type:* Form Trigger  
  - *Role:* Entry point; collects user input fields: Business Type, Location, Lead Number, Email Style (dropdown: Friendly, Professional, Simple).  
  - *Configuration:* Form titled "Lead Collect Tool" with required fields and a submit button labeled "Submit GO 🚀".  
  - *Inputs:* None (webhook trigger)  
  - *Outputs:* JSON with form data for downstream processing.  
  - *Failures:* Webhook errors, missing required fields.  
  - *Sticky Note1:* Explains this is Step 1: User fills the lead collection form.

---

#### 2.2 Business Data Acquisition

**Overview:**  
Uses Apify API to search for businesses matching the location and type provided by the user.

**Nodes Involved:**  
- HTTP Request  
- Sticky Note  

**Node Details:**  

- **HTTP Request**  
  - *Type:* HTTP Request  
  - *Role:* Sends JSON request to Apify API with parameters from form submission (location, business type, number of leads).  
  - *Configuration:* POST request with JSON body including search strings (business type), maximum leads, locationQuery, and various scraping options disabled except for reviews personal data.  
  - *Inputs:* JSON from form submission node.  
  - *Outputs:* JSON array of business data.  
  - *Failures:* API connectivity issues, rate limiting, malformed request.  
  - *Sticky Note:* Step 2 describes this as the step where Apify searches relevant businesses and website presence is checked later.

---

#### 2.3 Data Filtering

**Overview:**  
Filters businesses to keep only those that have a website field.

**Nodes Involved:**  
- Filter  
- Sticky Note2  

**Node Details:**  

- **Filter**  
  - *Type:* Filter  
  - *Role:* Filters incoming items to only those with a non-empty `website` field.  
  - *Configuration:* Condition checks existence of `website` string.  
  - *Inputs:* Business data array from HTTP Request.  
  - *Outputs:* Filtered list with websites only.  
  - *Failures:* Expression errors if `website` field missing in JSON.  
  - *Sticky Note2:* Describes Step 3: Email addresses will be extracted from these websites.

---

#### 2.4 Email Address Extraction

**Overview:**  
Calls OpenAI to extract the best single valid email address from the company website.

**Nodes Involved:**  
- Information Extractor  
- OpenAI Chat Model  
- Sticky Note2 (contextual)  

**Node Details:**  

- **Information Extractor**  
  - *Type:* LangChain Information Extractor  
  - *Role:* Uses a prompt to find one valid email address from the website data.  
  - *Configuration:* Prompt includes website URL and instructions to find the best email in ideal format.  
  - *Inputs:* Filtered business node output.  
  - *Outputs:* Extracted email address in JSON attribute `Email Address`.  
  - *Failures:* AI response errors, no email found, ambiguous or malformed email.  
  - *Version:* Requires LangChain nodes installed.  

- **OpenAI Chat Model**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* Provides AI language model backend for Information Extractor.  
  - *Configuration:* Model ID specified (not shown), default options.  
  - *Inputs:* Prompt from Information Extractor node.  
  - *Outputs:* AI-generated extraction data.  
  - *Failures:* API key/auth failures, rate limits, timeouts.

---

#### 2.5 Data Recording

**Overview:**  
Records the company details and extracted email into a Google Sheet.

**Nodes Involved:**  
- Append row in sheet  
- Sticky Note3  

**Node Details:**  

- **Append row in sheet**  
  - *Type:* Google Sheets (Append)  
  - *Role:* Appends new row with company data including Company Name, Category, Website, Phone Number, Email Address, Address.  
  - *Configuration:* Maps values from filtered and AI-enriched JSON fields to predefined columns.  
  - *Inputs:* Filtered and AI-extracted data.  
  - *Outputs:* Confirmation of appended row.  
  - *Failures:* Google Sheets API quota, permission errors, invalid document/sheet IDs.  
  - *Sticky Note3:* Step 4 - Businesses with emails are recorded in Google Sheets.

---

#### 2.6 Email Generation and Sending Loop

**Overview:**  
Iterates over each business to generate a personalized cold email using AI, send it via Gmail, update Google Sheets with status, and notify the user via Telegram.

**Nodes Involved:**  
- Loop Over Items  
- Wait  
- Information Extractor1  
- OpenAI Chat Model1  
- Edit Fields  
- If  
- Send a message (Gmail)  
- Append or update row in sheet  
- Send a text message (Telegram)  
- Sticky Note4, Sticky Note6, Sticky Note5  

**Node Details:**  

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Processes each business lead individually in batches (default batch size).  
  - *Inputs:* Output from Append row in sheet or Append or update row node.  
  - *Outputs:* Single business item per iteration.  
  - *Failures:* Loop interruptions, batch size misconfiguration.  

- **Wait**  
  - *Type:* Wait  
  - *Role:* Introduces delay between processing items to manage request pacing or API limits.  
  - *Inputs:* Each business item from loop.  
  - *Outputs:* Delayed item forward.  

- **Information Extractor1**  
  - *Type:* LangChain Information Extractor  
  - *Role:* Generates cold email subject and body using AI.  
  - *Configuration:* Prompt instructs to generate professional cold email for "Your name" agency based on Company Name, Business Type, and Email Style from form input.  
  - *Inputs:* Business item JSON.  
  - *Outputs:* Extracted `Mail Subject` and `Mail Body`.  
  - *Failures:* AI errors, prompt misinterpretation.  

- **OpenAI Chat Model1**  
  - *Type:* LangChain OpenAI Chat Model  
  - *Role:* AI backend for Information Extractor1.  
  - *Inputs:* Prompt from Information Extractor1.  
  - *Outputs:* AI email content.  

- **Edit Fields**  
  - *Type:* Set  
  - *Role:* Adds Send Time (current timestamp) and Email Address to the item JSON for tracking.  
  - *Inputs:* AI-generated mail data.  
  - *Outputs:* Enriched JSON with additional fields.  

- **If**  
  - *Type:* If Condition  
  - *Role:* Filters out invalid email addresses by checking if extracted email contains "@" symbol.  
  - *Inputs:* Enriched JSON.  
  - *Outputs:* True branch only proceeds with valid emails.  
  - *Failures:* Expression errors if `Email Address` missing.  

- **Send a message (Gmail)**  
  - *Type:* Gmail Node (Send Email)  
  - *Role:* Sends the cold email using Gmail SMTP with subject and body from AI output.  
  - *Configuration:* Sends to extracted email address, plain text email.  
  - *Inputs:* Valid emails only.  
  - *Outputs:* Confirmation of email sent.  
  - *Failures:* OAuth2 token expiration, Gmail API limits, invalid recipient address.  

- **Append or update row in sheet**  
  - *Type:* Google Sheets (Append or Update)  
  - *Role:* Updates the row for the business with "Cold Mail Status" as "✅" and the send time.  
  - *Configuration:* Matches on `Email Address` column to update existing row.  
  - *Inputs:* Confirmation from Send a message node.  
  - *Outputs:* Updated Google Sheet row.  
  - *Failures:* Sheet access issues, concurrency conflicts.  

- **Send a text message (Telegram)**  
  - *Type:* Telegram Node (Send Message)  
  - *Role:* Notifies the Telegram user about the email sent, including company name, email address, subject, and body.  
  - *Inputs:* Email sent info.  
  - *Outputs:* Confirmation of Telegram message sent.  
  - *Failures:* Telegram bot token/auth errors, network timeouts.  

- **Sticky Notes:**  
  - Sticky Note4: Describes the loop sending AI-generated emails one by one, filtering invalid emails.  
  - Sticky Note6: Details sending Telegram notifications for each email sent.  
  - Sticky Note5: Explains updating Google Sheets with cold email status.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                | Input Node(s)                  | Output Node(s)                               | Sticky Note                                                         |
|-------------------------|----------------------------------|-----------------------------------------------|-------------------------------|----------------------------------------------|--------------------------------------------------------------------|
| On form submission      | Form Trigger                     | Collect lead generation parameters             | None                          | HTTP Request                                 | Step 1: User fills the lead collection form                        |
| HTTP Request            | HTTP Request                    | Query Apify API for businesses                  | On form submission            | Filter                                       | Step 2: Apify searches businesses; websites filtered next          |
| Filter                  | Filter                         | Keep only businesses with websites              | HTTP Request                  | Information Extractor                         | Step 3: Email addresses are extracted from websites                |
| Information Extractor   | LangChain Information Extractor | Extract best email address from website         | Filter                        | If                                           | Step 3 context                                                     |
| OpenAI Chat Model       | LangChain OpenAI Chat Model     | AI backend for email address extraction          | Information Extractor         | Information Extractor                         |                                                                    |
| Append row in sheet     | Google Sheets (Append)          | Record business and email data                   | If                           | Loop Over Items                               | Step 4: Store data in Google Sheets                                |
| Loop Over Items         | SplitInBatches                  | Process each business lead one by one            | Append row in sheet           | Wait                                          | Step 5: Loop over leads for sending emails                         |
| Wait                    | Wait                           | Delay between processing items                    | Loop Over Items               | Information Extractor1                         |                                                                    |
| Information Extractor1  | LangChain Information Extractor | Generate cold email subject and body via AI      | Wait                         | Edit Fields                                   |                                                                    |
| OpenAI Chat Model1      | LangChain OpenAI Chat Model     | AI backend for cold email generation              | Information Extractor1        | Information Extractor1                         |                                                                    |
| Edit Fields             | Set                            | Add send time and email address for tracking     | Information Extractor1        | Send a message (Gmail) and If                 |                                                                    |
| If                      | If Condition                   | Validate email contains "@"                        | Edit Fields                  | Send a message (Gmail)                         | Step 5: Filters invalid emails                                     |
| Send a message (Gmail)  | Gmail                          | Send cold email                                   | If                           | Append or update row in sheet, Send a text message (Telegram) |                                                                    |
| Append or update row in sheet | Google Sheets (Append or Update) | Update cold email status and send time           | Send a message (Gmail)        | Loop Over Items                               | Step 6.2: Update Google Sheet with email status                   |
| Send a text message (Telegram) | Telegram                   | Notify Telegram user of sent email                | Send a message (Gmail)        | None                                          | Step 6.1: Notify user via Telegram                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Title: "Lead Collect Tool"  
   - Fields:  
     - Business Type (text, required)  
     - Location (text, required)  
     - Lead Number (number, required)  
     - Email Style (dropdown: Friendly, Professional, Simple)  
   - Configure webhook to receive form submission.

2. **Add HTTP Request Node**  
   - Connect from Form Trigger.  
   - Type: HTTP Request (POST)  
   - URL: Apify API endpoint (e.g., https://api.apify.com)  
   - Body (JSON): Include location, business type, lead number, disable unnecessary scraping features, enable review personal data. Use expressions to insert form data.  
   - Set Content-Type to application/json.

3. **Add Filter Node**  
   - Connect from HTTP Request.  
   - Condition: Keep only items where `website` field exists and is not empty.

4. **Add LangChain OpenAI Chat Model Node**  
   - Connect to Information Extractor node (next step).  
   - Configure credentials for OpenAI API.  
   - Select appropriate model (e.g., GPT-4 or GPT-3.5).

5. **Add Information Extractor Node**  
   - Connect from Filter output.  
   - Use the OpenAI Chat Model as language model.  
   - Prompt: Ask AI to extract the best single email address from the website URL.  
   - Define output attribute: `Email Address`.  

6. **Add If Condition Node**  
   - Connect from Information Extractor.  
   - Condition: Check if `Email Address` contains "@" symbol to validate email.  
   - True branch continues; False branch ignored or logged.

7. **Add Google Sheets Append Row Node**  
   - Connect from If (true branch).  
   - Map business data and extracted email address to columns: Company Name, Category, Website, Email Address, Phone Number, Address.  
   - Specify target Google Sheet and worksheet.  
   - Configure OAuth2 credentials for Google Sheets.

8. **Add SplitInBatches Node (Loop Over Items)**  
   - Connect from Google Sheets append node.  
   - Batch size default or as desired.

9. **Add Wait Node**  
   - Connect from SplitInBatches output.  
   - Configure delay (optional) to avoid hitting API limits.

10. **Add LangChain OpenAI Chat Model Node (for cold email generation)**  
    - Connect to Information Extractor1 node.  
    - Use the same OpenAI credentials.

11. **Add Information Extractor1 Node**  
    - Connect from Wait.  
    - Prompt: Generate cold email subject and body using company name, business type, and email style parameters.  
    - Output attributes: `Mail Subject`, `Mail Body`.

12. **Add Set Node (Edit Fields)**  
    - Connect from Information Extractor1.  
    - Add current timestamp as `Send Time` (format: dd-MM-yyyy (h:mm:s a)).  
    - Pass along `Email Address`.

13. **Add If Node (validate email again if needed)**  
    - Optional, can reuse earlier If node logic.

14. **Add Gmail Send Email Node**  
    - Connect from Edit Fields or If node.  
    - Configure Gmail OAuth2 credentials.  
    - Send to extracted `Email Address`.  
    - Subject and body from AI-generated mail subject and body.  
    - Email type: plain text.

15. **Add Google Sheets Append or Update Row Node**  
    - Connect from Gmail send node.  
    - Update row matching `Email Address`.  
    - Add `Cold Mail Status` as "✅" and `Send Time`.

16. **Add Telegram Send Message Node**  
    - Connect from Gmail send node.  
    - Configure Telegram bot credentials.  
    - Message text includes confirmation of email sent with company name, mail address, subject, and body.

17. **Connect Append or Update Row and Telegram nodes back to Loop Over Items**  
    - To continue processing next batch.

18. **Add Sticky Notes** for documentation clarity and step marking.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                                     |
|----------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| This workflow requires credentials for OpenAI, Google Sheets (OAuth2), Gmail (OAuth2), Telegram Bot API. | Credential setup is mandatory before running the workflow.        |
| The cold email prompt uses a specific style instruction to maintain professionalism and clarity. | Ensure prompt wording is consistent for quality emails.           |
| Telegram notifications allow instant feedback about emails sent, useful for monitoring.       | Bot token and chat ID must be valid and correctly configured.      |
| The workflow uses LangChain nodes for AI extraction and generation.                         | LangChain package must be installed and configured in n8n.        |
| Apify API usage may require an API key and subscription depending on the endpoint.            | Consult Apify API docs for rate limits and authentication.        |

---

**Disclaimer:**  
The text provided is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.