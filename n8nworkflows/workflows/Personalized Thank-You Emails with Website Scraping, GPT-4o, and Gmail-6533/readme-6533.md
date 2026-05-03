Personalized Thank-You Emails with Website Scraping, GPT-4o, and Gmail

https://n8nworkflows.xyz/workflows/personalized-thank-you-emails-with-website-scraping--gpt-4o--and-gmail-6533


# Personalized Thank-You Emails with Website Scraping, GPT-4o, and Gmail

### 1. Workflow Overview

This workflow automates the process of sending ultra-personalized thank-you emails immediately after a prospect submits an intake form. It targets agencies, consultants, freelancers, and B2B sales teams who want to create high-touch first impressions by referencing specific information from the prospect’s website. The workflow leverages website scraping, markdown conversion, AI-based content extraction and email copywriting (using GPT-4o models), a natural delay to simulate human typing, and Gmail for sending the email.

The workflow is logically divided into the following blocks:

- **1.1 Intake Reception and Website Scraping:** Triggered by a form submission, it scrapes the prospect’s website URL provided in the form.
- **1.2 Content Processing and AI Extraction:** Converts the scraped HTML to markdown and uses AI to extract plain text and a summary of the website.
- **1.3 AI-Powered Email Customization:** Uses the extracted website content to generate a short, casual, personalized thank-you email.
- **1.4 Delay and Email Sending:** Introduces a wait period to simulate a typing delay before sending the customized email via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Intake Reception and Website Scraping

**Overview:**  
This block listens for form submissions from Typeform, extracts the submitted website URL, and performs an HTTP request to scrape the website’s HTML content.

**Nodes Involved:**  
- Intake Form Submitted  
- Scrape Website  
- Markdown  
- Sticky Note (Intake form submitted -> Get website copy)

**Node Details:**  

- **Intake Form Submitted**  
  - *Type:* Typeform Trigger  
  - *Role:* Starts the workflow when a form (ID: LlUNhoPN) is submitted.  
  - *Configuration:* Connected to a Typeform account credential. Listens for submissions to the specified form.  
  - *Expressions:* Extracts form data including “What’s your website URL?” and prospect’s email and first name.  
  - *Input/Output:* No input; output is form submission JSON.  
  - *Edge Cases:* Webhook connectivity issues; missing or malformed URL in submission.  
  - *Sub-workflows:* None.

- **Scrape Website**  
  - *Type:* HTTP Request  
  - *Role:* Fetches the raw HTML content of the submitted website URL.  
  - *Configuration:* URL dynamically set from `$json["What's your website URL?"]`. No additional options configured.  
  - *Expressions:* URL parameter uses an expression to dynamically insert the form’s website URL.  
  - *Input/Output:* Input is form submission JSON; output is raw HTTP response containing HTML.  
  - *Edge Cases:* Invalid URL, request timeout, HTTP errors (404, 500), SSL issues.  
  - *Sub-workflows:* None.

- **Markdown**  
  - *Type:* Markdown Node  
  - *Role:* Converts the raw HTML content from the previous step into markdown format (text with minimal formatting).  
  - *Configuration:* Converts the raw HTML stored in `$json.data` into markdown.  
  - *Expressions:* Uses `={{ $json.data }}` to pass HTML content.  
  - *Input/Output:* Input is raw HTML; output is markdown text.  
  - *Edge Cases:* Empty or malformed HTML causing poor markdown conversion.  
  - *Sub-workflows:* None.

- **Sticky Note**  
  - *Role:* Visual annotation with the note “Intake form submitted -> Get website copy” indicating this block’s purpose.  
  - *Configuration:* Positioned for clarity in the editor, no runtime effect.

---

#### 2.2 Content Processing and AI Extraction

**Overview:**  
Transforms the markdown content into structured plain text and a one-line summary using an AI model fine-tuned to interpret web content.

**Nodes Involved:**  
- Website Plain Copy  
- Markdown (from previous block) connects here

**Node Details:**  

- **Website Plain Copy**  
  - *Type:* OpenAI Node (Langchain integration)  
  - *Role:* Processes markdown website content with GPT-4o-mini to extract plain text and a brief summary.  
  - *Configuration:*  
    - Model: `gpt-4o-mini`  
    - System prompt: Defines role as an intelligent web scraping assistant.  
    - Task prompt: Convert markdown to JSON with keys `plainTextWebsiteCopy` and `oneLineSummary`.  
    - Input: Injects markdown content with `=Markdown: {{ $json.data }}` expression.  
    - Output: JSON parsed response with the extracted fields.  
  - *Input/Output:* Input is markdown text; output is structured JSON with website copy and summary.  
  - *Edge Cases:* AI model failing to parse markdown, rate limits or API errors, malformed markdown input.  
  - *Requirements:* Valid OpenAI API credentials.  
  - *Sub-workflows:* None.

---

#### 2.3 AI-Powered Email Customization

**Overview:**  
Generates a short, casual, personalized thank-you email message referencing the prospect’s website data extracted earlier.

**Nodes Involved:**  
- Email Customization  
- Website Plain Copy (previous node)

- **Email Customization**  
  - *Type:* OpenAI Node (Langchain integration)  
  - *Role:* Creates a customized email snippet using GPT-4o based on the website’s plain text and summary.  
  - *Configuration:*  
    - Model: `gpt-4o`  
    - System prompt: Email copywriting assistant.  
    - User prompt: Uses a template emphasizing casual tone and short length (1-2 sentences).  
    - Variables: Injects `plainTextWebsiteCopy` and `oneLineSummary` from previous node’s output to inform the AI.  
    - Output: Email copy only, no additional text.  
  - *Input/Output:* Input is JSON from Website Plain Copy; output is generated email copy.  
  - *Edge Cases:* AI output empty or irrelevant, API quota exceeded, malformed inputs.  
  - *Requirements:* Valid OpenAI API credentials.  
  - *Sub-workflows:* None.

- **Sticky Note1**  
  - Annotation: "Write a 'customized' thank you email," visually indicating the purpose of this block.

---

#### 2.4 Delay and Email Sending

**Overview:**  
Simulates a natural typing delay before sending the customized email via Gmail.

**Nodes Involved:**  
- Wait  
- Gmail  
- Email Customization (previous node)

- **Wait**  
  - *Type:* Wait Node  
  - *Role:* Pauses workflow execution for 250 seconds (~4 minutes 10 seconds) to simulate a natural delay before sending the email.  
  - *Configuration:* Fixed wait time set to 250 seconds.  
  - *Input/Output:* Input receives customized email text; output passes data forward after delay.  
  - *Edge Cases:* Workflow timeouts or interruptions, unusually long waiting may trigger workflow time limits.  
  - *Sub-workflows:* None.

- **Gmail**  
  - *Type:* Gmail Node  
  - *Role:* Sends the personalized thank-you email to the prospect’s email address.  
  - *Configuration:*  
    - Recipient email dynamically set from form submission `Email` field.  
    - Subject: "Thanks for reaching out" (static).  
    - Message body: Uses an expression to compose a friendly email starting with “Hey [First Name], I just got your form submission.” followed by the AI-generated message content and a polite sign-off.  
    - Credential: Uses OAuth2 Gmail credential named “builtbyabdul”.  
    - Options: Disables attribution footer appended by Gmail node.  
  - *Expressions:* Multiple expressions extract first name and AI message content to build message body.  
  - *Input/Output:* Input is message content from Wait node; output is email send status.  
  - *Edge Cases:* Authentication errors, Gmail API rate limits, invalid email addresses, message length limits.  
  - *Sub-workflows:* None.

---

### 3. Summary Table

| Node Name             | Node Type                  | Functional Role                             | Input Node(s)        | Output Node(s)       | Sticky Note                                                                                                              |
|-----------------------|----------------------------|---------------------------------------------|----------------------|----------------------|--------------------------------------------------------------------------------------------------------------------------|
| Intake Form Submitted  | Typeform Trigger           | Starts workflow on form submission          | None                 | Scrape Website       | # Send a personalized thank-you email after form submission using website insights (shared note covers multiple nodes)   |
| Scrape Website        | HTTP Request               | Scrapes submitted website URL                | Intake Form Submitted | Markdown             | # Send a personalized thank-you email after form submission using website insights                                        |
| Markdown              | Markdown Node              | Converts raw HTML to markdown                 | Scrape Website       | Website Plain Copy   | # Send a personalized thank-you email after form submission using website insights                                        |
| Website Plain Copy    | OpenAI (Langchain)         | Extracts plain text and summary from markdown | Markdown             | Email Customization  | # Send a personalized thank-you email after form submission using website insights                                        |
| Email Customization   | OpenAI (Langchain)         | Generates personalized email content          | Website Plain Copy   | Wait                 | Write a "customized" thank you email                                                                                     |
| Wait                  | Wait Node                  | Introduces delay to simulate typing           | Email Customization  | Gmail                |                                                                                                                          |
| Gmail                 | Gmail Node                 | Sends personalized thank-you email            | Wait                 | None                 |                                                                                                                          |
| Sticky Note           | Sticky Note                | Annotation: Intake form submitted -> Get website copy | None             | None                 | Intake form submitted -> Get website copy                                                                                |
| Sticky Note1          | Sticky Note                | Annotation: Write a "customized" thank you email | None               | None                 | Write a "customized" thank you email                                                                                     |
| Sticky Note2          | Sticky Note                | Workflow overview and instructions           | None                 | None                 | # Send a personalized thank-you email after form submission using website insights (Detailed workflow description)       |
| Sticky Note4          | Sticky Note                | Author’s contact and branding information    | None                 | None                 | Hey, I'm Abdul 👋 I build growth systems for consultants & agencies. https://www.builtbyabdul.com/ builtbyabdul@gmail.com |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node**  
   - Add a *Typeform Trigger* node named `Intake Form Submitted`.  
   - Set the `formId` parameter to your Typeform form ID (e.g., "LlUNhoPN").  
   - Attach your Typeform API credentials.

2. **Add HTTP Request Node for Website Scraping**  
   - Add an *HTTP Request* node named `Scrape Website`.  
   - For the URL, use the expression: `={{ $json["What's your website URL?"] }}` to dynamically fetch the submitted URL.  
   - Leave other settings default.

3. **Add Markdown Node to Convert HTML to Markdown**  
   - Add a *Markdown* node named `Markdown`.  
   - Set the input to: `={{ $json.data }}` to convert the HTTP response HTML.  
   - No extra options are required.

4. **Add OpenAI Node for Website Text Extraction**  
   - Add an *OpenAI* (Langchain) node named `Website Plain Copy`.  
   - Select model `gpt-4o-mini`.  
   - Configure the system prompt to instruct the assistant as a web scraping helper.  
   - Input prompt to instruct the assistant to output a JSON with keys `plainTextWebsiteCopy` and `oneLineSummary` based on markdown content.  
   - Use expression to pass markdown: `"=Markdown: {{ $json.data }}"`.  
   - Enable JSON output parsing.  
   - Attach your OpenAI API credentials.

5. **Add OpenAI Node for Email Generation**  
   - Add another *OpenAI* (Langchain) node named `Email Customization`.  
   - Select model `gpt-4o`.  
   - Configure system prompt as an intelligent email copywriter.  
   - User prompt should instruct to generate a short, casual email snippet referencing the website data from the previous node, using this template:  
     `" (CompanyName) looks great, love your (uniqueValueProp/something interesting)."`  
   - Use expressions to insert:  
     - `{{ $json.message.content.plainTextWebsiteCopy }}`  
     - `{{ $json.message.content.oneLineSummary }}`  
   - Attach OpenAI API credentials.

6. **Add Wait Node to Simulate Delay**  
   - Add a *Wait* node named `Wait`.  
   - Set the wait time to 250 seconds (approx. 4 minutes 10 seconds).  
   - Connect from `Email Customization`.

7. **Add Gmail Node to Send Email**  
   - Add a *Gmail* node named `Gmail`.  
   - Set `Send To` to: `={{ $('Intake Form Submitted').item.json.Email }}`.  
   - Set `Subject` to `"Thanks for reaching out"`.  
   - Compose the message body using expressions:  
     ```
     Hey {{ $('Intake Form Submitted').item.json['First name'].split(" ").first() }}, I just got your form submission.<br><br>
     {{ $json.message.content }} Thanks for getting in touch, looking forward to chatting with you later. <br>
     Feel free to reach out if you have any questions:)<br><br>
     Best,<br>
     Abdul
     ```  
   - Disable attribution footer in options.  
   - Attach your Gmail OAuth2 credentials.

8. **Connect the Nodes in Order:**  
   - `Intake Form Submitted` → `Scrape Website` → `Markdown` → `Website Plain Copy` → `Email Customization` → `Wait` → `Gmail`.

9. **Add Sticky Notes for Documentation (Optional but Recommended):**  
   - Add sticky notes to annotate the purpose of each block as per the original workflow for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                   | Context or Link                                               |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| This workflow impresses leads with personalized thank-you emails referencing their website, automating high-touch follow-ups for agencies, consultants, freelancers, and B2B sales teams.                                                                        | Workflow purpose and target audience                           |
| For setup, OpenAI credentials, Typeform API access, and Gmail OAuth2 credentials are required.                                                                                                                                                                  | Credential and integration requirements                        |
| The delay (250 seconds) simulates natural typing to increase email authenticity; adjust as needed.                                                                                                                                                              | Simulation of human behavior                                   |
| The AI prompts are customizable for tone, length, and style to fit brand voice or audience preferences.                                                                                                                                                         | AI prompt customization                                       |
| Author: Abdul — builds growth systems for consultants & agencies. Website: https://www.builtbyabdul.com/ Email: builtbyabdul@gmail.com                                                                                                                        | Author contact and branding                                    |
| Example use case provided: Prospect submits form with website URL, receives a customized thank-you email referencing clean UX or unique mission — demonstrating the workflow’s real-world application.                                                          | Use case example                                              |
| Potential extensions: Add CRM integration, lead scoring, Slack alerts, or fallback logic if scraping fails.                                                                                                                                                      | Suggested workflow enhancements                               |

---

**Disclaimer:**  
The text provided is exclusively from an automated workflow built with n8n, respecting all content policies. It contains no illegal, offensive, or protected material. All data processed is lawful and public.