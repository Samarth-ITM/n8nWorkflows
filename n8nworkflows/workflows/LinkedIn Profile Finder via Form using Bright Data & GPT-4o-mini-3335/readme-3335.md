LinkedIn Profile Finder via Form using Bright Data & GPT-4o-mini

https://n8nworkflows.xyz/workflows/linkedin-profile-finder-via-form-using-bright-data---gpt-4o-mini-3335


# LinkedIn Profile Finder via Form using Bright Data & GPT-4o-mini

### 1. Workflow Overview

This workflow automates the process of finding LinkedIn profiles for a given person based on their full name and company, using a user-submitted form. It leverages Bright Data to scrape Google search results and OpenAI’s GPT-4o-mini model to parse and analyze the data. The workflow then generates a personalized follow-up email with insights and outreach recommendations, which is sent to a predefined email address.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input via a web form.
- **1.2 LinkedIn Profile Search:** Constructs a Google search query, scrapes results via Bright Data, extracts HTML content, and parses LinkedIn profile data using GPT-4o-mini.
- **1.3 Profile Matching and Filtering:** Filters parsed results to find matching LinkedIn profiles and handles cases where no profile is found.
- **1.4 Company Data Lookup:** Performs a separate Google search for the company, scrapes and parses company-related data.
- **1.5 Content Generation:** Merges profile and company data, then uses GPT-4o-mini to generate a buyer persona analysis and personalized outreach content.
- **1.6 Email Delivery and Confirmation:** Sends the generated email and displays a confirmation message to the user.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures prospect data (person’s full name and company) via a web form and triggers the workflow.

- **Nodes Involved:**  
  - When User Completes Form

- **Node Details:**  
  - **When User Completes Form**  
    - Type: Form Trigger  
    - Configuration:  
      - Webhook path: `search-user`  
      - Form fields: "Person Fullname" (required), "Person's company" (required)  
      - Button label: "Get References"  
      - Response mode: last node output  
      - Form description explains the purpose and outcome  
    - Inputs: HTTP request from form submission  
    - Outputs: JSON containing form data  
    - Edge cases: Missing required fields will block submission; bot requests ignored  
    - Version: 2.2  

---

#### 1.2 LinkedIn Profile Search

- **Overview:**  
  Builds a Google search URL targeting LinkedIn profiles, scrapes the search results using Bright Data, extracts HTML content, and parses the results with GPT-4o-mini to identify LinkedIn profiles.

- **Nodes Involved:**  
  - Edit Url LinkedIn  
  - Get LinkedIn Entry on Google  
  - Extract Body and Title from Website  
  - Parse Google Results

- **Node Details:**  
  - **Edit Url LinkedIn**  
    - Type: Set  
    - Role: Constructs Google search URL for LinkedIn profiles using person’s full name and company  
    - Key expression:  
      `https://www.google.com/search?q=site:linkedin.com/in+{{ encodeURIComponent($json["Person Fullname"].trim() + " " + $json["Person's company"].trim()) }}`  
    - Inputs: Form data  
    - Outputs: JSON with `google_search` URL  
    - Version: 3.4  

  - **Get LinkedIn Entry on Google**  
    - Type: Bright Data (web scraping)  
    - Role: Scrapes Google search results page for LinkedIn profiles  
    - Configuration:  
      - URL from `google_search` field  
      - Zone: `web_unlocker1`  
      - Country: US  
      - Format: JSON  
    - Credentials: BrightData API  
    - Inputs: URL from previous node  
    - Outputs: Scraped search results JSON  
    - Edge cases: Possible scraping blocks, rate limits, or invalid credentials  

  - **Extract Body and Title from Website**  
    - Type: HTML Extract  
    - Role: Extracts `<title>` and `<body>` HTML content from scraped page  
    - Configuration: Extract `title` and `body` CSS selectors, trim values  
    - Inputs: Bright Data scraped JSON  
    - Outputs: JSON with extracted HTML content  
    - Edge cases: Missing or malformed HTML may cause extraction failure  

  - **Parse Google Results**  
    - Type: OpenAI (GPT-4o-mini)  
    - Role: Parses extracted HTML to identify LinkedIn profiles (link, fullname, position, company) and marks matches based on user input  
    - Configuration:  
      - Model: GPT-4o-mini  
      - System prompt instructs extraction of LinkedIn profiles and matching logic  
      - Input text: extracted HTML body  
      - JSON output enabled  
    - Credentials: OpenAI API  
    - Inputs: Extracted HTML content  
    - Outputs: Parsed results with `results` array including `match` property  
    - Edge cases: API timeouts, parsing errors, or incomplete data  

---

#### 1.3 Profile Matching and Filtering

- **Overview:**  
  Splits parsed results, filters for matching profiles, limits to one profile, and branches workflow based on whether a profile is found.

- **Nodes Involved:**  
  - Extract Parsed Results  
  - Get only Matching Profiles  
  - Limit to 1 Profile  
  - LinkedIn Profile is Found?  
  - Form Not Found

- **Node Details:**  
  - **Extract Parsed Results**  
    - Type: Split Out  
    - Role: Splits the `message.content.results` array into individual items for filtering  
    - Inputs: Parsed results JSON  
    - Outputs: Individual profile entries  
    - Version: 1  

  - **Get only Matching Profiles**  
    - Type: Filter  
    - Role: Filters profiles where `match` property equals `true`  
    - Condition: `$json.match === true`  
    - Inputs: Individual profiles  
    - Outputs: Matching profiles only  
    - Version: 2.2  

  - **Limit to 1 Profile**  
    - Type: Limit  
    - Role: Restricts output to the first matching profile  
    - Inputs: Filtered profiles  
    - Outputs: Single profile or none  
    - Version: 1  

  - **LinkedIn Profile is Found?**  
    - Type: If  
    - Role: Checks if a profile exists (non-empty JSON)  
    - Condition: JSON object exists  
    - True branch: proceeds to merge data  
    - False branch: triggers "Form Not Found" node  
    - Version: 2.2  

  - **Form Not Found**  
    - Type: Form (completion)  
    - Role: Sends a response to user indicating no matching profile was found  
    - Response text includes person’s full name and company from form data  
    - Inputs: False branch from If node  
    - Outputs: HTTP response to user  
    - Version: 1  

---

#### 1.4 Company Data Lookup

- **Overview:**  
  Performs a Google search for the company, scrapes results via Bright Data, extracts HTML content, and parses company-related data with GPT-4o-mini.

- **Nodes Involved:**  
  - Edit Company Search  
  - Get Company on Google  
  - Extract Body and Title from Website1  
  - Parse Google Results for Company  
  - Split Out

- **Node Details:**  
  - **Edit Company Search**  
    - Type: Set  
    - Role: Constructs Google search URL for the company name  
    - Key expression:  
      `https://www.google.com/search?q={{ encodeURIComponent($json["Person's company"].trim()) }}`  
    - Inputs: Form data  
    - Outputs: JSON with `google_search` URL  
    - Version: 3.4  

  - **Get Company on Google**  
    - Type: Bright Data  
    - Role: Scrapes Google search results page for company data  
    - Configuration similar to LinkedIn scraping node  
    - Credentials: BrightData API  
    - Inputs: URL from previous node  
    - Outputs: Scraped search results JSON  
    - Edge cases: Same as LinkedIn scraping node  

  - **Extract Body and Title from Website1**  
    - Type: HTML Extract  
    - Role: Extracts `<title>` and `<body>` HTML content from scraped company page  
    - Inputs: Bright Data scraped JSON  
    - Outputs: Extracted HTML content  
    - Version: 1.2  

  - **Parse Google Results for Company**  
    - Type: OpenAI (GPT-4o-mini)  
    - Role: Parses extracted HTML to find the first matching company entry and outputs it as content  
    - Configuration:  
      - Model: GPT-4o-mini  
      - System prompt instructs to find first matching company entry  
      - Input text: extracted HTML body  
      - JSON output enabled  
    - Credentials: OpenAI API  
    - Inputs: Extracted HTML content  
    - Outputs: Parsed company data in `message.content`  
    - Edge cases: API errors, incomplete data  

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the parsed company content for merging  
    - Inputs: Parsed company data  
    - Outputs: Individual company data entries  
    - Version: 1  

---

#### 1.5 Content Generation

- **Overview:**  
  Merges the selected LinkedIn profile and company data, then generates a personalized follow-up email content with buyer persona analysis and outreach recommendations using GPT-4o-mini.

- **Nodes Involved:**  
  - Merge  
  - Create a Followup for Company and Person

- **Node Details:**  
  - **Merge**  
    - Type: Merge  
    - Role: Combines LinkedIn profile data and company data by position (side-by-side)  
    - Inputs: True branch LinkedIn profile and company data split out  
    - Outputs: Combined JSON object  
    - Version: 3.1  

  - **Create a Followup for Company and Person**  
    - Type: OpenAI (GPT-4o-mini)  
    - Role: Generates buyer persona analysis and outreach strategy as raw HTML styled with Tailwind CSS  
    - Configuration:  
      - Model: GPT-4o-mini  
      - System prompt instructs to analyze data and output recommendations with concrete outreach steps  
      - Input text: JSON stringified merged data  
      - JSON output enabled  
    - Credentials: OpenAI API  
    - Inputs: Merged data  
    - Outputs: HTML content in `message.content.content`  
    - Edge cases: API timeouts, malformed input data  

---

#### 1.6 Email Delivery and Confirmation

- **Overview:**  
  Sends the generated personalized email to a predefined address and shows a confirmation message to the user.

- **Nodes Involved:**  
  - Send Email  
  - Form Email Sent

- **Node Details:**  
  - **Send Email**  
    - Type: Email Send  
    - Role: Sends an HTML email with the generated follow-up content  
    - Configuration:  
      - Subject: "Next followup"  
      - To and From email: `miquel@n8nhackers.com` (hardcoded)  
      - HTML body: from GPT-generated content  
    - Credentials: SMTP account  
    - Inputs: Generated HTML content  
    - Outputs: Email sent confirmation  
    - Edge cases: SMTP auth failures, invalid email addresses  

  - **Form Email Sent**  
    - Type: Form (completion)  
    - Role: Sends a thank-you confirmation message to the user after email is sent  
    - Configuration:  
      - Completion title: "Thank you!"  
      - Completion message: "We have sent you an email"  
    - Inputs: After email sent  
    - Outputs: HTTP response to user  
    - Version: 1  

---

### 3. Summary Table

| Node Name                      | Node Type                 | Functional Role                          | Input Node(s)                     | Output Node(s)                    | Sticky Note                                                                                      |
|--------------------------------|---------------------------|----------------------------------------|----------------------------------|---------------------------------|------------------------------------------------------------------------------------------------|
| When User Completes Form        | Form Trigger              | Captures user input from form          | -                                | Edit Url LinkedIn, Edit Company Search |                                                                                                |
| Edit Url LinkedIn               | Set                       | Builds Google search URL for LinkedIn | When User Completes Form          | Get LinkedIn Entry on Google     |                                                                                                |
| Get LinkedIn Entry on Google    | Bright Data               | Scrapes Google search results          | Edit Url LinkedIn                 | Extract Body and Title from Website |                                                                                                |
| Extract Body and Title from Website | HTML Extract           | Extracts HTML title and body            | Get LinkedIn Entry on Google      | Parse Google Results             |                                                                                                |
| Parse Google Results            | OpenAI (GPT-4o-mini)      | Parses LinkedIn profiles from HTML     | Extract Body and Title from Website | Extract Parsed Results          |                                                                                                |
| Extract Parsed Results          | Split Out                 | Splits parsed LinkedIn profiles        | Parse Google Results              | Get only Matching Profiles       |                                                                                                |
| Get only Matching Profiles      | Filter                    | Filters profiles matching user input   | Extract Parsed Results            | Limit to 1 Profile               |                                                                                                |
| Limit to 1 Profile              | Limit                     | Limits to one matching profile          | Get only Matching Profiles        | LinkedIn Profile is Found?       |                                                                                                |
| LinkedIn Profile is Found?      | If                        | Checks if profile found                  | Limit to 1 Profile                | Merge (true), Form Not Found (false) |                                                                                                |
| Form Not Found                 | Form (completion)          | Responds if no profile found             | LinkedIn Profile is Found? (false) | -                             |                                                                                                |
| Edit Company Search             | Set                       | Builds Google search URL for company    | When User Completes Form          | Get Company on Google            |                                                                                                |
| Get Company on Google           | Bright Data               | Scrapes Google search results for company | Edit Company Search             | Extract Body and Title from Website1 |                                                                                                |
| Extract Body and Title from Website1 | HTML Extract           | Extracts HTML title and body for company | Get Company on Google           | Parse Google Results for Company |                                                                                                |
| Parse Google Results for Company | OpenAI (GPT-4o-mini)     | Parses company data from HTML           | Extract Body and Title from Website1 | Split Out                    |                                                                                                |
| Split Out                      | Split Out                 | Splits parsed company data              | Parse Google Results for Company | Merge                          |                                                                                                |
| Merge                         | Merge                     | Combines LinkedIn profile and company data | LinkedIn Profile is Found? (true), Split Out | Create a Followup for Company and Person |                                                                                                |
| Create a Followup for Company and Person | OpenAI (GPT-4o-mini) | Generates personalized outreach content | Merge                          | Send Email                     |                                                                                                |
| Send Email                    | Email Send                | Sends personalized email                 | Create a Followup for Company and Person | Form Email Sent               |                                                                                                |
| Form Email Sent               | Form (completion)          | Shows confirmation message to user      | Send Email                      | -                               |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node**  
   - Type: Form Trigger  
   - Webhook path: `search-user`  
   - Form title: "Sales prospecting"  
   - Form fields:  
     - "Person Fullname" (required, placeholder: "Complete the fullname")  
     - "Person's company" (required, placeholder: "Complete the company")  
   - Button label: "Get References"  
   - Response mode: last node output  
   - Description: Explain the purpose and outcome to user  

2. **Add a Set node named "Edit Url LinkedIn"**  
   - Purpose: Build Google search URL for LinkedIn profiles  
   - Add field `google_search` with value:  
     `https://www.google.com/search?q=site:linkedin.com/in+{{ encodeURIComponent($json["Person Fullname"].trim() + " " + $json["Person's company"].trim()) }}`  
   - Connect from Form Trigger node  

3. **Add Bright Data node "Get LinkedIn Entry on Google"**  
   - Purpose: Scrape Google search results  
   - URL: `={{ $json.google_search }}`  
   - Zone: `web_unlocker1`  
   - Country: US  
   - Format: JSON  
   - Credentials: BrightData API account  
   - Connect from "Edit Url LinkedIn"  

4. **Add HTML Extract node "Extract Body and Title from Website"**  
   - Purpose: Extract `<title>` and `<body>` from scraped page  
   - Extraction keys: `title` (selector: `title`), `body` (selector: `body`)  
   - Trim values enabled  
   - Connect from Bright Data node  

5. **Add OpenAI node "Parse Google Results"**  
   - Model: GPT-4o-mini  
   - System prompt:  
     ```
     Extract Linkedin profiles from google results (link, fullname, position, company if possible). 
     Return a results property with all the parsed results including a property "match" if user matches the data entry values "{{ $('When User Completes Form').item.json["Person Fullname"].trim() }} {{ $('When User Completes Form').item.json["Person Position"].trim() }} {{ $('When User Completes Form').item.json["Person's company"].trim() }}"
     ```  
   - Input text: `{{ $json.body }}`  
   - JSON output enabled  
   - Credentials: OpenAI API  
   - Connect from HTML Extract node  

6. **Add Split Out node "Extract Parsed Results"**  
   - Field to split out: `message.content.results`  
   - Connect from OpenAI node  

7. **Add Filter node "Get only Matching Profiles"**  
   - Condition: `$json.match === true`  
   - Connect from Split Out node  

8. **Add Limit node "Limit to 1 Profile"**  
   - Limit to 1 item  
   - Connect from Filter node  

9. **Add If node "LinkedIn Profile is Found?"**  
   - Condition: JSON object exists (check if profile found)  
   - Connect from Limit node  

10. **Add Form node "Form Not Found"**  
    - Operation: Completion  
    - Respond with: showText  
    - Response text:  
      `We didn't found a person for "{{ $('When User Completes Form').item.json["Person Fullname"] }} {{ $('When User Completes Form').item.json["Person Fullname"] }} {{ $('When User Completes Form').item.json["Person's company"] }}"`  
    - Connect from If node (false branch)  

11. **Add Set node "Edit Company Search"**  
    - Purpose: Build Google search URL for company  
    - Field `google_search`:  
      `https://www.google.com/search?q={{ encodeURIComponent($json["Person's company"].trim()) }}`  
    - Connect from Form Trigger node  

12. **Add Bright Data node "Get Company on Google"**  
    - Same configuration as LinkedIn Bright Data node  
    - URL: `={{ $json.google_search }}`  
    - Credentials: BrightData API  
    - Connect from "Edit Company Search"  

13. **Add HTML Extract node "Extract Body and Title from Website1"**  
    - Same extraction keys as previous HTML node  
    - Connect from Bright Data company node  

14. **Add OpenAI node "Parse Google Results for Company"**  
    - Model: GPT-4o-mini  
    - System prompt:  
      ```
      Get first entry matching company {{ $('When User Completes Form').item.json["Person's company"] }}
      Output first entry data in a content property
      ```  
    - Input text: `{{ $json.body }}`  
    - JSON output enabled  
    - Credentials: OpenAI API  
    - Connect from HTML Extract company node  

15. **Add Split Out node "Split Out"**  
    - Field to split out: `message.content`  
    - Connect from OpenAI company node  

16. **Add Merge node "Merge"**  
    - Mode: Combine by position  
    - Connect true branch of If node (LinkedIn profile found) to first input  
    - Connect Split Out company node to second input  

17. **Add OpenAI node "Create a Followup for Company and Person"**  
    - Model: GPT-4o-mini  
    - System prompt:  
      ```
      Use data to analyze as a buyer persona. Find the best approach to connect for future champion in his company. Give recommendations and a concrete outreach steps.
      Output report as raw html in a property called content. Use tailwind for styles.
      ```  
    - Input text: `{{ JSON.stringify($json) }}`  
    - JSON output enabled  
    - Credentials: OpenAI API  
    - Connect from Merge node  

18. **Add Email Send node "Send Email"**  
    - Subject: "Next followup"  
    - To: `miquel@n8nhackers.com` (hardcoded)  
    - From: `miquel@n8nhackers.com` (hardcoded)  
    - HTML body: `{{ $json.message.content.content }}`  
    - Credentials: SMTP account  
    - Connect from OpenAI follow-up node  

19. **Add Form node "Form Email Sent"**  
    - Operation: Completion  
    - Completion title: "Thank you!"  
    - Completion message: "We have sent you an email"  
    - Connect from Email Send node  

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                   |
|-------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow template created by Miquel Colomer and n8nhackers                                      | Creator LinkedIn: https://www.linkedin.com/in/miquelcolomersalas/                               |
| Support and customization consulting available via email                                        | Contact: support@n8nhackers.com                                                                 |
| Uses Tailwind CSS for styling the generated email content                                       | Styling framework for HTML email output                                                         |
| BrightData credentials required for Google scraping                                            | https://brightdata.com/                                                                          |
| OpenAI GPT-4o-mini model used for parsing and content generation                               | OpenAI API: https://platform.openai.com/                                                       |
| SMTP credentials required for email sending                                                    | Ensure valid SMTP account configured in n8n                                                      |

---

This documentation provides a complete and detailed reference for understanding, reproducing, and modifying the LinkedIn Profile Finder workflow using Bright Data and GPT-4o-mini in n8n.