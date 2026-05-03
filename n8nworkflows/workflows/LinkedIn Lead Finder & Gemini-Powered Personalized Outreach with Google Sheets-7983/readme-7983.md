LinkedIn Lead Finder & Gemini-Powered Personalized Outreach with Google Sheets

https://n8nworkflows.xyz/workflows/linkedin-lead-finder---gemini-powered-personalized-outreach-with-google-sheets-7983


# LinkedIn Lead Finder & Gemini-Powered Personalized Outreach with Google Sheets

### 1. Workflow Overview

This workflow automates a personalized LinkedIn lead generation and outreach process powered by Google Gemini AI and Google Sheets integration. It targets professionals or marketers who want to find LinkedIn profiles of companies or professionals based on specific keywords and purposes, craft personalized outreach messages, and maintain structured records in Google Sheets.

The workflow logic can be grouped into the following blocks:

- **1.1 Input Reception:** Receiving user input keywords and contact purpose through a form trigger.
- **1.2 AI-Powered Search Query Generation:** Using Google Gemini (PaLM) to generate Boolean search strings tailored for LinkedIn searches.
- **1.3 LinkedIn Profile Search:** Executing Google Custom Search API calls to find LinkedIn company or professional profiles.
- **1.4 Data Processing & Looping:** Splitting search results and processing each company/profile individually.
- **1.5 AI-Powered Message Drafting:** Using Google Gemini AI to create personalized outreach messages for each identified LinkedIn profile.
- **1.6 Google Sheets Management:** Appending or updating company/profile data and messages in Google Sheets.
- **1.7 Notification:** Sending an email notification when the auto-writing messages are completed.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
Collects user input from a web form including keywords to find companies or professionals on LinkedIn and the purpose of contact.

- **Nodes Involved:**  
  - Form submit

- **Node Details:**

  - *Form submit*  
    - Type: Form Trigger  
    - Role: Entry point capturing user input via a web form.  
    - Configuration: Form titled "1.0 S_LG_Find LinkedIn Accounts by keywords and Write personal message for contact" with two fields: "Keywords to find Company / Professional" and "Purpose of Contact".  
    - Inputs: Webhook call from user form submission.  
    - Outputs: Passes JSON containing user inputs downstream.  
    - Edge cases: Missing or malformed input fields may result in empty or invalid data. Should validate inputs externally or downstream.

---

#### 2.2 AI-Powered Search Query Generation

- **Overview:**  
Generates optimized Boolean search strings for LinkedIn based on keywords and purpose using Google Gemini AI.

- **Nodes Involved:**  
  - Create boolean search strings

- **Node Details:**

  - *Create boolean search strings*  
    - Type: Google Gemini AI (PaLM) node  
    - Role: Generates Boolean search strings specifically including "site:linkedin.com" for Google searches.  
    - Configuration: Uses model "models/gemini-2.5-flash". The prompt instructs the AI to create search queries based on the form fields `Keywords to find Company / Professional` and `Purpose of Contact`.  
    - Expressions: Input fields referenced via expressions `{{ $json['Keywords to find Company / Professional'] }}` and `{{ $json['Purpose of Contact'] }}`.  
    - Credentials: Uses a configured Google Gemini API credential.  
    - Output: JSON with Boolean query string.  
    - Edge cases: API failures, rate limits, malformed prompts.  
    - Failure mode: Empty or invalid query string could break downstream search.

---

#### 2.3 LinkedIn Profile Search

- **Overview:**  
Performs a Google Custom Search API call to find LinkedIn profiles using the generated Boolean query string.

- **Nodes Involved:**  
  - Get Linkedin Company

- **Node Details:**

  - *Get Linkedin Company*  
    - Type: HTTP Request  
    - Role: Calls Google Custom Search API to retrieve LinkedIn search results.  
    - Configuration:  
      - URL: `https://www.googleapis.com/customsearch/v1`  
      - Query parameters: API key, CSE ID (`cx`), query string (`q`) from AI node output, number of results (20), language (`hl=vi`), geographic location (`gl=vn`).  
    - Expressions: Query string expression `={{ $json.content.parts[0].text }}` uses Boolean string generated previously.  
    - Edge cases: API key invalid, quota exceeded, no search results, location parameters mismatch.  
    - Notes: Language and region parameters should be adjusted for user location (see Sticky Note4).  
    - Output: JSON with search results containing title, link, description, etc.

---

#### 2.4 Data Processing & Looping

- **Overview:**  
Processes search results by splitting them into individual items and looping over them for further processing.

- **Nodes Involved:**  
  - Split Out  
  - Loop Over Items

- **Node Details:**

  - *Split Out*  
    - Type: Split Out  
    - Role: Splits the array of search results (items) into individual items to process one at a time.  
    - Configuration: Field to split out: "items".  
    - Input: Search results JSON.  
    - Output: Each item separately downstream.

  - *Loop Over Items*  
    - Type: Split In Batches  
    - Role: Processes each individual LinkedIn profile item in batches (default batch size).  
    - Configuration: Default options, no explicit batch size set.  
    - Input: Individual items from Split Out node.  
    - Output: Each item passed to two parallel nodes: "Send email" and "Append row in sheet".  
    - Edge cases: Large batch sizes may cause timeouts or rate limiting.

---

#### 2.5 AI-Powered Message Drafting

- **Overview:**  
Generates personalized outreach messages for each LinkedIn profile using Gemini AI, based on the purpose and company information.

- **Nodes Involved:**  
  - Append row in sheet  
  - Writing message  
  - Update sheet

- **Node Details:**

  - *Append row in sheet*  
    - Type: Google Sheets Append or Update  
    - Role: Adds or updates a row with company info (name, URL, description) extracted from search result item.  
    - Configuration:  
      - Maps Google search result fields to sheet columns:  
        - `name` ← title  
        - `linkedin_url` ← link  
        - `des` ← og:description from page metadata  
      - Uses Google Sheets OAuth2 credentials.  
      - Operation: appendOrUpdate by matching `linkedin_url`.  
    - Output: Passes enriched JSON downstream to "Writing message".

  - *Writing message*  
    - Type: Google Gemini AI (PaLM) node  
    - Role: Drafts personalized outreach message using company name, description, and contact purpose.  
    - Configuration: Model "models/gemini-1.5-flash".  
    - Prompt includes placeholders if info missing.  
    - Expressions: Pulls `Purpose of Contact` from the original form (`$('Form submit').item.json['Purpose of Contact']`) and company details from current item.  
    - Output: Message text.  
    - Credentials: Google Gemini API.  

  - *Update sheet*  
    - Type: Google Sheets Update  
    - Role: Updates existing Google Sheet row with the generated outreach message.  
    - Configuration:  
      - Matches rows by `linkedin_url`.  
      - Updates `message` and `row_number` columns, keeps other columns intact.  
      - Uses Google Sheets OAuth2 credentials.  
    - Output: Loops back to "Loop Over Items" node for further processing.  
    - Edge cases: Failure to match row, Google Sheets API errors.

---

#### 2.6 Notification

- **Overview:**  
Sends email notification once all processing of the batch is completed.

- **Nodes Involved:**  
  - Send email

- **Node Details:**

  - *Send email*  
    - Type: Email Send  
    - Role: Notifies a predefined email address that all auto-writing messages have been completed.  
    - Configuration:  
      - Subject: "Your auto writing message be completed"  
      - To: info@example.com  
      - From: admin@example.com  
      - HTML body: "[Link to sheet]" (a placeholder link, presumably to the Google Sheet)  
    - Credentials: SMTP credentials configured.  
    - Triggered after each batch processed.  
    - Edge cases: SMTP auth failure, email sending limits.

---

#### 2.7 Sticky Notes (Documentation & Setup Hints)

- Sticky Note3: Explains Google Sheets column mapping for clarity.
- Sticky Note4: Advises adjusting `hl` (language) and `gl` (geolocation) parameters in Google Custom Search for localization.
- Sticky Note5: Workflow overview summarized in steps.
- Sticky Note6: Lists required setup tasks including credentials and API key configuration.

---

### 3. Summary Table

| Node Name               | Node Type                 | Functional Role                                | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                                                                                   |
|-------------------------|---------------------------|-----------------------------------------------|-------------------------|-----------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Form submit             | Form Trigger              | Captures keywords and purpose via web form    | (Webhook)               | Create boolean search strings | Sticky Note5: Workflow overview summarized in steps                                                                                                          |
| Create boolean search strings | Google Gemini AI (PaLM) | Generates Boolean LinkedIn search queries      | Form submit             | Get Linkedin Company          | Sticky Note6: Setup Required - Credentials and API key configuration                                                                                          |
| Get Linkedin Company     | HTTP Request             | Performs Google Custom Search for LinkedIn URLs | Create boolean search strings | Split Out                    | Sticky Note4: Adjust hl & gl parameters for location                                                                                                         |
| Split Out               | Split Out                 | Splits search results into individual items    | Get Linkedin Company     | Loop Over Items              |                                                                                                                                                               |
| Loop Over Items         | Split In Batches          | Processes each LinkedIn profile in batches     | Split Out, Update sheet  | Send email, Append row in sheet |                                                                                                                                                               |
| Append row in sheet     | Google Sheets Append/Update | Saves LinkedIn profile info to Google Sheets   | Loop Over Items          | Writing message             | Sticky Note3: Google Sheets column mapping explanation                                                                                                       |
| Writing message         | Google Gemini AI (PaLM)   | Drafts personalized outreach messages          | Append row in sheet      | Update sheet                |                                                                                                                                                               |
| Update sheet            | Google Sheets Update      | Updates row with generated outreach message    | Writing message          | Loop Over Items             |                                                                                                                                                               |
| Send email              | Email Send                | Sends notification email on completion         | Loop Over Items          | (End)                      |                                                                                                                                                               |
| Sticky Note3            | Sticky Note               | Explains Google Sheets mapping                  |                         |                             | ## Google Sheets Mapping - name → Company/Person name, linkedin_url → LinkedIn URL, des → description, message → AI outreach text                           |
| Sticky Note4            | Sticky Note               | Notes on localization parameters for search    |                         |                             | ### Update hl & gl to fit with your location                                                                                                                 |
| Sticky Note5            | Sticky Note               | Workflow overview summary                        |                         |                             | ## Overview - Collect inputs, generate queries, search LinkedIn, save to sheets, generate messages, update sheets, notify email                              |
| Sticky Note6            | Sticky Note               | Setup instructions                              |                         |                             | ## Setup Required - Google Sheets OAuth2, Gemini API, SMTP, Custom Search Engine ID and API key, location parameters                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node ("Form submit")**  
   - Type: Form Trigger  
   - Configure form title: "1.0 S_LG_Find LinkedIn Accounts by keywords and Write personal message for contact"  
   - Add two form fields:  
     - "Keywords to find Company / Professional"  
     - "Purpose of Contact"  
   - Save and activate webhook.

2. **Add Google Gemini AI node ("Create boolean search strings")**  
   - Model: `models/gemini-2.5-flash`  
   - Prompt: Instruct AI to generate Boolean LinkedIn search strings based on inputs:  
     ```
     =You are a tool that generates Boolean search strings for Google, in order to search for companies or professionals on LinkedIn.

     # Task:
     - Based on the keyword and the purpose, generate a suitable keyword string for searching on LinkedIn.
     - Only return the keyword string, without any explanation or additional information.
     - Always add site:linkedin.com to the keyword string.

     # Input:
     Keyword: {{ $json['Keywords to find Company / Professional'] }}
     Purpose: {{ $json['Purpose of Contact'] }}
     ```
   - Connect output of "Form submit" to this node.  
   - Set Google Gemini API credentials.

3. **Add HTTP Request node ("Get Linkedin Company")**  
   - Method: GET  
   - URL: `https://www.googleapis.com/customsearch/v1`  
   - Query parameters:  
     - `key`: Your Google API key  
     - `cx`: Your Custom Search Engine ID  
     - `q`: Expression: `={{ $json.content.parts[0].text }}` (Boolean string from previous node)  
     - `num`: "20" (max results)  
     - `hl`: Language code (e.g., "vi")  
     - `gl`: Geo code (e.g., "vn")  
   - Connect output of "Create boolean search strings" to this node.

4. **Add Split Out node ("Split Out")**  
   - Field to split out: "items" (from search results array)  
   - Connect output of "Get Linkedin Company" to this node.

5. **Add Split In Batches node ("Loop Over Items")**  
   - Use default batch size or configure as needed.  
   - Connect output of "Split Out" to this node.

6. **Add Google Sheets Append or Update node ("Append row in sheet")**  
   - Operation: Append or Update  
   - Document ID: Your Google Sheet ID  
   - Sheet name: e.g., gid=0 or "Sheet1"  
   - Mapping columns:  
     - `name`: `={{ $json.title }}`  
     - `linkedin_url`: `={{ $json.link }}`  
     - `des`: `={{ $json.pagemap.metatags[0]["og:description"] }}`  
   - Matching column: `linkedin_url`  
   - Credentials: Google Sheets OAuth2  
   - Connect one output from "Loop Over Items" to this node.

7. **Add Google Gemini AI node ("Writing message")**  
   - Model: `models/gemini-1.5-flash`  
   - Prompt:  
     ```
     =You are an AI that drafts outreach messages.

     # Requirements
     Based on the purpose: {{ $('Form submit').item.json['Purpose of Contact'] }}
     and the company information:  
     name: {{ $json.name }}  
     description: {{ $json.des }}  

     Write a suitable and personalized outreach message for that specific company.

     Only return the outreach message, without any explanation or additional text.
     If some information is missing, use placeholders.
     ```
   - Connect output of "Append row in sheet" to this node.  
   - Set Google Gemini API credentials.

8. **Add Google Sheets Update node ("Update sheet")**  
   - Operation: Update  
   - Document ID and sheet same as above  
   - Matching column: `linkedin_url`  
   - Map columns:  
     - `message`: AI-generated message from "Writing message" node  
     - `row_number`: 0 (or dynamically from sheet API if available)  
   - Credentials: Google Sheets OAuth2  
   - Connect output of "Writing message" to this node.

9. **Connect output of "Update sheet" back to "Loop Over Items"**  
   - This forms the batch processing loop.

10. **Add Email Send node ("Send email")**  
    - Subject: "Your auto writing message be completed"  
    - To: info@example.com (change as needed)  
    - From: admin@example.com (change as needed)  
    - HTML content: "[Link to sheet]" (replace with actual Google Sheet link)  
    - Credentials: SMTP configured  
    - Connect second output of "Loop Over Items" to this node.

11. **Add Sticky Notes for documentation** (optional)  
    - Add notes for Google Sheets mapping, setup instructions, overview, and localization reminders.

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                             |
|------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Google Sheets column mapping: name, linkedin_url, des, message                                         | Explains how LinkedIn search results map to sheet columns                                                  |
| Adjust `hl` and `gl` parameters in Google Custom Search API for your language and geographic region   | See Sticky Note4                                                                                           |
| Workflow overview: input collection, AI query generation, search, data saving, AI message writing, notification | See Sticky Note5                                                                                           |
| Setup required: Google Sheets OAuth2, Google Gemini API credentials, SMTP credentials, Google Custom Search API key and CSE ID | See Sticky Note6                                                                                           |
| Use placeholders in AI message drafting if company info is missing                                    | AI prompt includes fallback instructions                                                                   |
| This workflow requires valid API keys and credentials for: Google Gemini, Google Sheets OAuth2, SMTP, Google Custom Search API |                                                                                                            |

---

**Disclaimer:** The text above is exclusively derived from an automated workflow created with n8n, complying strictly with applicable content policies and containing no illegal, offensive, or protected elements. All manipulated data is legal and public.