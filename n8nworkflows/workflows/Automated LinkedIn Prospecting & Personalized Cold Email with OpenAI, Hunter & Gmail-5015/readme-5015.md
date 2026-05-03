Automated LinkedIn Prospecting & Personalized Cold Email with OpenAI, Hunter & Gmail

https://n8nworkflows.xyz/workflows/automated-linkedin-prospecting---personalized-cold-email-with-openai--hunter---gmail-5015


# Automated LinkedIn Prospecting & Personalized Cold Email with OpenAI, Hunter & Gmail

### 1. Workflow Overview

This workflow automates LinkedIn prospecting and personalized cold email generation leveraging OpenAI, Hunter.io, Google Search, and Gmail. It is designed to transform a job description or prospect criteria into Boolean search strings for LinkedIn profiles, scrape relevant LinkedIn URLs and workplace contexts via Google search, extract detailed contact information using AI, find emails via Hunter.io, and generate tailored cold outreach emails. The workflow stores all results in Google Sheets and drafts cold emails in Gmail for review and sending.

**Target use cases:**  
- Recruiting professionals sourcing LinkedIn profiles matching job descriptions  
- Sales or business development teams performing personalized cold outreach  
- Automated lead generation with enriched contact data and email drafts

**Logical blocks:**  
- **1.1 Input Reception & Boolean Search String Generation**: Receives chat input, uses OpenAI to generate a Boolean search string for LinkedIn profile search on Google.  
- **1.2 Google Search & LinkedIn URL Extraction**: Performs authenticated Google searches, extracts LinkedIn URLs and workplace context snippets from HTML.  
- **1.3 Contact Detail Extraction & Email Lookup**: Uses OpenAI to parse LinkedIn info into structured contact details, then queries Hunter.io for email addresses.  
- **1.4 Personalized Cold Email Generation**: Generates personalized cold emails using OpenAI based on contact and workplace context.  
- **1.5 Data Storage & Email Drafting**: Appends all results to Google Sheets and drafts cold emails in Gmail.  
- **1.6 Pagination & Looping Control**: Manages Google search pagination with wait nodes and conditional looping until desired results count is reached.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception & Boolean Search String Generation

**Overview:**  
Receives user chat input (e.g., job description), generates a precise Boolean search string tailored for LinkedIn profile searches via OpenAI GPT-4o-mini. Creates a new Google Sheet for storing results.

**Nodes involved:**  
- When chat message received  
- Generate a Boolean Search String  
- Create a new sheet  
- Columns to add: linkedin_url, first_name, last_name, email, context, domain

**Node Details:**  

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Entry trigger node to start workflow on received chat message  
  - Configuration: Default webhook, no special parameters  
  - Input: External chat message  
  - Output: Passes chat input to next node  
  - Failures: Missing or malformed chat input  

- **Generate a Boolean Search String**  
  - Type: LangChain OpenAI node  
  - Role: Generates Boolean search string for Google LinkedIn profile search  
  - Configuration: Uses GPT-4o-mini model with system prompt instructing to generate Boolean string from job description input; outputs JSON with search_string and sheet_name  
  - Expressions: `={{ $json.chatInput }}` inputs user chat message  
  - Output: JSON with Boolean string, e.g. `site:linkedin.com/in [search terms]`  
  - Failures: OpenAI API errors, malformed response, missing API key  

- **Create a new sheet**  
  - Type: Google Sheets node  
  - Role: Creates a new Google Sheet using sheet_name from previous node with current timestamp  
  - Configuration: Document ID set to user's Google Drive; title dynamically from Boolean search node output + timestamp  
  - Credentials: Google Sheets OAuth2  
  - Output: Sheet metadata including title  
  - Failures: Google Sheets API auth errors, quota exceeded  

- **Columns to add: linkedin_url, first_name, last_name, email, context, domain**  
  - Type: Code node  
  - Role: Initializes empty columns schema for new sheet append operations  
  - Output: JSON object with empty fields to define columns  
  - Failures: None expected  

---

#### 2.2 Google Search & LinkedIn URL Extraction

**Overview:**  
Executes Google searches with the Boolean string; extracts LinkedIn URLs and workplace context from search result HTML, including handling pagination and rate limits.

**Nodes involved:**  
- Add columns to new sheet  
- set page number for google search  
- Google Boolean Search  
- Extracts all linkedin urls and workplace context from the google http response  
- If desired results not reached  
- Wait  
- Adds 10 to start - Go to next page

**Node Details:**  

- **Add columns to new sheet**  
  - Type: Google Sheets  
  - Role: Appends initialized columns to the newly created sheet  
  - Configuration: Uses sheet title from "Create a new sheet" node  
  - Output: Confirmation of append  
  - Failures: Google Sheets quota or auth  

- **set page number for google search**  
  - Type: Code node  
  - Role: Initializes search pagination starting index to 0  
  - Output: `{ start: 0 }` for Google search query param  
  - Failures: None expected  

- **Google Boolean Search**  
  - Type: HTTP Request  
  - Role: Sends authenticated Google search request with Boolean string and pagination start index  
  - Configuration:  
    - URL: https://www.google.com/search  
    - Query param `q` from Boolean search string  
    - Query param `start` for pagination  
    - Header includes User-Agent and cookie header for authenticated search (cookie string must be manually updated as per Sticky Note4)  
  - Credentials: HTTP Header Auth (cookie-based)  
  - Output: Raw HTML content of Google search results  
  - Failures: HTTP errors, auth failure, IP blocking, rate limiting  

- **Extracts all linkedin urls and workplace context from the google http response**  
  - Type: Code node  
  - Role: Parses Google HTML to extract unique LinkedIn profile URLs and associated workplace context snippets from HTML divs (selectors YrbPuc and VwiC3b)  
  - Output: Array of objects with `linkedin_url` and `workplace_context`  
  - Failures: Parsing errors if Google changes HTML structure, empty results  

- **If desired results not reached**  
  - Type: If node  
  - Role: Checks if pagination should continue: start index < 3 (i.e., up to 30 results) and other conditions  
  - Failures: Logic errors if conditions misconfigured  
  - Note: The default max pages is 3; can be adjusted per Sticky Note2  

- **Wait**  
  - Type: Wait node  
  - Role: Waits 5 seconds between requests to avoid Google rate limiting  
  - Failures: None expected  

- **Adds 10 to start - Go to next page**  
  - Type: Code node  
  - Role: Increments start index by 10 to request next page of Google results  
  - Output: New `{ start }` value  
  - Failures: None expected  

---

#### 2.3 Contact Detail Extraction & Email Lookup

**Overview:**  
Uses OpenAI to extract structured contact details (first name, last name, domain name, context) from LinkedIn URLs and workplace context. Then uses Hunter.io API to find corresponding email addresses.

**Nodes involved:**  
- Extract Contact Details  
- Extracts fname, lname, domainname  
- Hunter

**Node Details:**  

- **Extract Contact Details**  
  - Type: LangChain OpenAI node  
  - Role: For each LinkedIn URL and workplace context, extracts structured fields: linkedin_url, first_name, last_name, domain_name, meaningful_context  
  - Model: chatgpt-4o-latest  
  - Prompt: Explicit instructions to parse and infer domain names and meaningful context from workplace text  
  - Output: JSON with extracted contact info  
  - Failures: OpenAI API errors, insufficient context  

- **Extracts fname, lname, domainname**  
  - Type: Code node  
  - Role: Parses AI output from previous node to extract and normalize fields, handling JSON or regex fallback extraction  
  - Output: Items with fields first_name, last_name, domain_name, linkedin_url, meaningful_context  
  - Failures: Parsing failures on malformed AI output  

- **Hunter**  
  - Type: Hunter.io node  
  - Role: Finds email address given domain, first name, last name  
  - Configuration: Domain, firstname, lastname from prior node outputs  
  - Credentials: Hunter API key (free tier 25 requests/month)  
  - Output: Email address if found  
  - Failures: API quota exceeded, invalid domain, no email found  

---

#### 2.4 Personalized Cold Email Generation

**Overview:**  
Generates a formal, personalized cold email using OpenAI GPT-4o-latest based on profile details and workplace context. Emails are split into subject and body. Drafts are created in Gmail.

**Nodes involved:**  
- Personalized Cold-Email Generator  
- Extract subject and email body  
- Gmail  
- Appends the results to the sheet

**Node Details:**  

- **Personalized Cold-Email Generator**  
  - Type: LangChain OpenAI node  
  - Role: Creates a concise, formal cold outreach email tailored to prospect’s LinkedIn profile and context  
  - Model: chatgpt-4o-latest  
  - Prompt: Detailed system prompt with sender background, tone, email structure, and length instructions  
  - Input: first_name, last_name, meaningful_context, linkedin_url  
  - Output: Email text including subject and body concatenated  
  - Failures: OpenAI API errors, prompt misconfiguration  

- **Extract subject and email body**  
  - Type: Code node  
  - Role: Splits the generated email content into subject and email body by parsing on double newlines and removing "Subject:" prefix  
  - Output: JSON fields `subject` and `emailBody`  
  - Failures: Parsing errors if format changes  

- **Gmail**  
  - Type: Gmail node  
  - Role: Drafts cold email in Gmail drafts folder  
  - Configuration: Uses extracted subject and emailBody, sends to found email address  
  - Credentials: Gmail OAuth2  
  - Output: Draft creation confirmation  
  - Failures: Gmail API auth failure, quota limits  

- **Appends the results to the sheet**  
  - Type: Google Sheets  
  - Role: Saves all contact and email data (linkedin_url, first_name, last_name, email, context, email_subject, email_body) to Google Sheet  
  - Output: Append confirmation  
  - Failures: Google Sheets API errors  

---

#### 2.5 Pagination & Looping Control

**Overview:**  
Controls the iterative Google search page fetching, stopping when desired number of results reached or pages exhausted.

**Nodes involved:**  
- If desired results not reached  
- Wait  
- Adds 10 to start - Go to next page  
- set page number for google search

**Node Details:**  

- Already covered above in section 2.2.

---

### 3. Summary Table

| Node Name                                         | Node Type                  | Functional Role                                      | Input Node(s)                           | Output Node(s)                          | Sticky Note                                                                                                     |
|--------------------------------------------------|----------------------------|-----------------------------------------------------|---------------------------------------|----------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| When chat message received                        | LangChain Chat Trigger     | Workflow entry: receives chat input                  | -                                     | Generate a Boolean Search String        | Under "Credential to connect with" add your openAI API key. Find at: https://platform.openai.com/settings/organization/api-keys |
| Generate a Boolean Search String                  | LangChain OpenAI           | Generates Boolean search string from job description| When chat message received             | Create a new sheet                     |                                                                                                                 |
| Create a new sheet                                | Google Sheets              | Creates new Google Sheet for results                  | Generate a Boolean Search String       | Columns to add: linkedin_url, first_name, last_name, email, context, domain | Please change the name of the sheet to a sheet in your Google Drive                                            |
| Columns to add: linkedin_url, first_name, last_name, email, context, domain | Code                       | Initializes columns schema                            | Create a new sheet                     | Add columns to new sheet                |                                                                                                                 |
| Add columns to new sheet                          | Google Sheets              | Appends columns to new sheet                          | Columns to add: linkedin_url, first_name, last_name, email, context, domain | set page number for google search       |                                                                                                                 |
| set page number for google search                 | Code                       | Initializes Google search pagination index           | Add columns to new sheet               | If desired results not reached          | For the first condition: {{ $json.start }} is less than 50, so change "50" to your desired number of results. Each loop fetches the next page, returning 10 results per iteration. |
| If desired results not reached                    | If                         | Checks if more Google pages to fetch                  | set page number for google search, Adds 10 to start - Go to next page | Wait (if true), end loop (if false)   |                                                                                                                 |
| Wait                                             | Wait                       | Waits 5 seconds between Google requests               | If desired results not reached         | Google Boolean Search                   | Waits 5 seconds to avoid rate limiting by Google. While it's unlikely you'll be rate-limited since you're authenticated with your cookie, this is just a precaution. |
| Adds 10 to start - Go to next page                | Code                       | Increments pagination start index                      | If desired results not reached          | If desired results not reached          |                                                                                                                 |
| Google Boolean Search                            | HTTP Request               | Performs authenticated Google search with Boolean string | Wait                                 | Extracts all linkedin urls and workplace context from the google http response | Get this Cookie-Editor. https://chromewebstore.google.com/detail/cookie-editor/hlkenndednhfkekhgcdicdfddnkalmdm\nDo a google search --> click this extension --> Export --> Header string.\nThen, open this node --> under Header Auth --> edit --> and under cookie value paste in your header string.  \nThis is to perform an authenticated google search. |
| Extracts all linkedin urls and workplace context from the google http response | Code                       | Parses HTML to extract LinkedIn URLs and workplace context | Google Boolean Search                 | Extract Contact Details                 |                                                                                                                 |
| Extract Contact Details                          | LangChain OpenAI           | Extracts structured contact details from LinkedIn URLs and context | Extracts all linkedin urls and workplace context from the google http response | Extracts fname, lname, domainname       | This basically calls ChatGPT to infer the place of work from workplace context. This helps form the domain name for Hunter. |
| Extracts fname, lname, domainname                | Code                       | Parses AI output to structured contact fields          | Extract Contact Details                 | Personalized Cold-Email Generator       |                                                                                                                 |
| Hunter                                           | Hunter.io                  | Finds email address given domain and names             | Extract subject and email body (via chain) | Gmail, Appends the results to the sheet | Create an account on Hunter.io and add the API key here. 25 uses a month in the free model.                      |
| Personalized Cold-Email Generator                | LangChain OpenAI           | Generates formal, personalized cold email               | Extracts fname, lname, domainname       | Extract subject and email body          | Please change the system prompt to add information about yourself (it currently has my details). Also, change the system prompt for the task you are interested in (job request /sales demo etc) |
| Extract subject and email body                    | Code                       | Splits generated email into subject and body            | Personalized Cold-Email Generator       | Hunter                                  |                                                                                                                 |
| Appends the results to the sheet                  | Google Sheets              | Saves all prospect and email data                        | Hunter                                 | Adds 10 to start - Go to next page      | Stores all the cold emails in a Google Sheet                                                                  |
| Gmail                                            | Gmail                      | Drafts cold email in Gmail drafts folder                 | Hunter                                 | -                                      | Please add your Gmail credentials. You will find the cold email drafts in your Gmail drafts folder for review. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n named "LinkedIn Agent v2".**

2. **Add node:** *When chat message received* (LangChain Chat Trigger)  
   - No special config.  
   - Set as webhook trigger to receive chat input (job description).  

3. **Add node:** *Generate a Boolean Search String* (LangChain OpenAI)  
   - Model: GPT-4o-mini  
   - System prompt: Instruct to generate Boolean string for LinkedIn profile search from input text (see prompt in workflow).  
   - Input message: `={{ $json.chatInput }}` from trigger.  
   - Output: JSON with `search_string` and `sheet_name`.  
   - Connect trigger → this node.

4. **Add node:** *Create a new sheet* (Google Sheets)  
   - Operation: Create new spreadsheet  
   - Title: `={{ $('Generate a Boolean Search String').item.json.choices[0].message.content.sheet_name + ' ' + $now }}`  
   - Document ID: Your Google Drive folder ID or leave blank for default.  
   - Credentials: Set Google Sheets OAuth2 credentials.  
   - Connect Boolean Search → this node.

5. **Add node:** *Columns to add: linkedin_url, first_name, last_name, email, context, domain* (Code)  
   - JS code: returns JSON with empty column keys for sheet append schema.  
   - Connect Create new sheet → this node.

6. **Add node:** *Add columns to new sheet* (Google Sheets)  
   - Operation: Append  
   - Columns: Auto-map input, schema defined from code output.  
   - Sheet Name: From Create new sheet node title.  
   - Document ID: Same as Create new sheet.  
   - Credentials: Use same Google Sheets OAuth2.  
   - Connect Code node → this node.

7. **Add node:** *set page number for google search* (Code)  
   - JS code: returns `{ start: 0 }` to initialize pagination.  
   - Connect Add columns to new sheet → this node.

8. **Add node:** *Google Boolean Search* (HTTP Request)  
   - URL: `https://www.google.com/search`  
   - Query parameters:  
     - `q` = `={{ $('Generate a Boolean Search String').first().json.choices[0].message.content.search_string }}`  
     - `start` = `={{ $json.start }}`  
   - Headers:  
     - `User-Agent`: Use modern browser user agent string  
     - `cookie`: Use cookie string from Google authenticated session (see Sticky Note4 for how to get cookie string)  
   - Auth: HTTP Header Auth with above headers  
   - Connect Wait node (below) → this node.

9. **Add node:** *Extracts all linkedin urls and workplace context from the google http response* (Code)  
   - JS code: Parses HTML to extract LinkedIn URLs and workplace context snippets.  
   - Connect Google Boolean Search → this node.

10. **Add node:** *Extract Contact Details* (LangChain OpenAI)  
    - Model: chatgpt-4o-latest  
    - Prompt: For each LinkedIn URL and workplace context, extract linkedin_url, first_name, last_name, domain_name, meaningful_context in English.  
    - Connect LinkedIn extraction → this node.

11. **Add node:** *Extracts fname, lname, domainname* (Code)  
    - JS code: Parses AI output JSON or uses regex fallback to extract and clean fields.  
    - Connect Extract Contact Details → this node.

12. **Add node:** *Hunter* (Hunter.io API)  
    - Operation: emailFinder  
    - Parameters:  
      - domain: `={{ $('Extracts fname, lname, domainname').item.json.domain_name }}`  
      - firstname, lastname: same from prior node  
    - Credentials: Hunter API key  
    - Connect Extract subject and email body node (below) → this node.

13. **Add node:** *Personalized Cold-Email Generator* (LangChain OpenAI)  
    - Model: chatgpt-4o-latest  
    - System prompt: Formal, personalized cold outreach email writing instructions with sender’s background info (customize accordingly).  
    - Input messages: pass first_name, last_name, meaningful_context, linkedin_url.  
    - Connect Extracts fname, lname, domainname → this node.

14. **Add node:** *Extract subject and email body* (Code)  
    - JS code: Splits email generated by previous node into subject and email body fields.  
    - Connect Personalized Cold-Email Generator → this node.

15. **Add node:** *Appends the results to the sheet* (Google Sheets)  
    - Operation: Append  
    - Columns: linkedin_url, first_name, last_name, domain_name, email, context, email_subject, email_body  
    - Sheet Name: Same as created sheet  
    - Document ID: Same Google Drive document ID  
    - Credentials: Google Sheets OAuth2  
    - Connect Hunter → this node.

16. **Add node:** *Gmail* (Gmail node)  
    - Operation: Draft (resource = draft)  
    - Send To: `={{ $json.email }}`  
    - Subject: `={{ $('Extract subject and email body').item.json.subject }}`  
    - Message: `={{ $('Extract subject and email body').item.json.emailBody }}`  
    - Credentials: Gmail OAuth2  
    - Connect Hunter → this node.

17. **Add node:** *If desired results not reached* (If)  
    - Condition: Continue if  
      - `start` < 3 (adjustable for max pages)  
      - Other filters as needed  
    - Connect set page number for google search → this node.  
    - True branch → Wait node  
    - False branch → end loop

18. **Add node:** *Wait*  
    - Duration: 5 seconds  
    - Connect If node (true branch) → Wait → Google Boolean Search (loop)

19. **Add node:** *Adds 10 to start - Go to next page* (Code)  
    - JS: increment start by 10  
    - Connect Appends the results to the sheet → Adds 10 to start → If desired results not reached (loop control)

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                                          |
|------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------|
| Get this Cookie-Editor Chrome extension to export authenticated Google search cookie string for header auth:           | https://chromewebstore.google.com/detail/cookie-editor/hlkenndednhfkekhgcdicdfddnkalmdm                                   |
| Under "Credential to connect with" add your OpenAI API key.                                                             | https://platform.openai.com/settings/organization/api-keys                                                                |
| Change the Google Sheet name to a sheet in your Google Drive to avoid permission errors.                                 | -                                                                                                                        |
| Wait 5 seconds after each Google search request to avoid rate limiting, even if authenticated.                         | -                                                                                                                        |
| Hunter.io free account allows 25 API calls per month. Add your API key in the Hunter node.                              | https://hunter.io                                                                                                         |
| Customize the OpenAI system prompt for cold email generation to reflect your personal background and outreach goals.   | -                                                                                                                        |
| Gmail OAuth2 credentials are required for drafting cold emails automatically.                                           | -                                                                                                                        |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.