Search LinkedIn companies, Score with AI and add them to Google Sheet CRM

https://n8nworkflows.xyz/workflows/search-linkedin-companies--score-with-ai-and-add-them-to-google-sheet-crm-3904


# Search LinkedIn companies, Score with AI and add them to Google Sheet CRM

### 1. Workflow Overview

This workflow automates the process of searching for companies on LinkedIn, scoring their fit for a specific product or service using AI, and adding qualified companies to a Google Sheets CRM while avoiding duplicates. It targets sales teams, business developers, and marketers aiming to build a qualified prospect database efficiently.

The workflow is logically divided into three main blocks:

- **1.1 LinkedIn Company Search:** Initiates a search for companies on LinkedIn using the Ghost Genius API based on user-defined criteria such as target keywords, location, and company size.

- **1.2 Company Data Processing:** Processes each retrieved company individually by fetching detailed company information, filtering based on quality indicators (e.g., follower count, website presence), and checking for duplicates in the CRM.

- **1.3 AI Scoring and CRM Integration:** Uses OpenAI to score companies on their likelihood to be interested in the user’s product or service, then appends the scored companies to a Google Sheets CRM with rate-limit respect.

---

### 2. Block-by-Block Analysis

#### 2.1 LinkedIn Company Search

- **Overview:**  
  This block performs a paginated search on LinkedIn companies using the Ghost Genius API, based on variables set by the user (target audience, location, company size). It outputs a list of companies matching the criteria.

- **Nodes Involved:**  
  - Start  
  - Set Variables  
  - Search Companies  
  - Extract Company Data  
  - Process Each Company

- **Node Details:**

  - **Start**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Inputs: None  
    - Outputs: Triggers "Set Variables" node.  
    - Edge Cases: None.

  - **Set Variables**  
    - Type: Set  
    - Role: Stores user-defined parameters such as target industry, company size, location ID, product/service description, and positive/negative indicators for AI scoring.  
    - Configuration: Static strings and codes for company size and location, customizable by the user.  
    - Key Variables:  
      - "Your target" (e.g., "Growth Marketing Agency")  
      - Company size code (e.g., "C" for 11-50 employees)  
      - Location ID (numeric code from Ghost Genius location tool)  
      - Product/service description  
      - Positive and negative indicators for AI scoring  
    - Inputs: Triggered by "Start"  
    - Outputs: Feeds "Search Companies" node.  
    - Edge Cases: Misconfiguration leads to poor search results.

  - **Search Companies**  
    - Type: HTTP Request  
    - Role: Queries Ghost Genius API for companies matching the criteria with pagination (max 3 pages, 2-second interval between requests).  
    - Configuration:  
      - URL: Ghost Genius search endpoint  
      - Query Parameters: Keywords, locations, company size from "Set Variables"  
      - Authentication: Header Auth with Bearer token  
      - Pagination: Controlled to avoid exceeding API limits and max 1000 results  
    - Inputs: From "Set Variables"  
    - Outputs: JSON response with company list to "Extract Company Data"  
    - Edge Cases: API rate limits, network errors, empty responses, authentication failures.

  - **Extract Company Data**  
    - Type: Split Out  
    - Role: Splits the array of companies from the search response into individual items for processing.  
    - Configuration: Field to split out is "data" from API response.  
    - Inputs: From "Search Companies"  
    - Outputs: Individual company items to "Process Each Company"  
    - Edge Cases: Empty data arrays.

  - **Process Each Company**  
    - Type: Split In Batches  
    - Role: Processes companies one by one to manage API rate limits downstream.  
    - Configuration: Default batch size (1), no delay configured here (delay handled in HTTP Request node).  
    - Inputs: From "Extract Company Data"  
    - Outputs: To "Get Company Info" or loops back if company invalid  
    - Edge Cases: Batch processing errors, empty batches.

---

#### 2.2 Company Data Processing

- **Overview:**  
  This block retrieves detailed company information from the Ghost Genius API, filters companies based on follower count and website presence, and checks if the company already exists in the CRM to avoid duplicates.

- **Nodes Involved:**  
  - Get Company Info  
  - Filter Valid Companies  
  - Check If Company Exists  
  - Is New Company?  
  - Process Each Company (loop back)

- **Node Details:**

  - **Get Company Info**  
    - Type: HTTP Request  
    - Role: Fetches detailed company info using the company LinkedIn URL.  
    - Configuration:  
      - URL: Ghost Genius company endpoint  
      - Query parameter: "url" set to current company’s LinkedIn URL  
      - Authentication: Header Auth with Bearer token  
      - Batching: 1 request per 2 seconds to respect rate limits  
      - Retry on failure enabled  
    - Inputs: From "Process Each Company"  
    - Outputs: Detailed company data to "Filter Valid Companies"  
    - Edge Cases: API timeouts, invalid URLs, auth errors.

  - **Filter Valid Companies**  
    - Type: If  
    - Role: Filters companies that have a non-empty website and more than 200 followers.  
    - Configuration:  
      - Condition 1: website field is not empty  
      - Condition 2: followers_count > 200  
      - Both conditions must be true to proceed  
    - Inputs: From "Get Company Info"  
    - Outputs:  
      - True branch: to "Check If Company Exists"  
      - False branch: loops back to "Process Each Company" (skip invalid)  
    - Edge Cases: Missing fields, zero followers, malformed data.

  - **Check If Company Exists**  
    - Type: Google Sheets (Lookup)  
    - Role: Checks if the company ID already exists in the Google Sheet CRM to prevent duplicates.  
    - Configuration:  
      - Lookup column: "ID"  
      - Lookup value: current company’s ID  
      - Sheet: "Companies" in specified Google Sheet document  
      - Always outputs data to allow conditional check  
    - Inputs: From "Filter Valid Companies" (true branch)  
    - Outputs: To "Is New Company?"  
    - Edge Cases: Google Sheets API errors, connectivity issues.

  - **Is New Company?**  
    - Type: If  
    - Role: Determines if the company is new by checking if lookup result is empty.  
    - Configuration:  
      - Condition: lookup result empty (no existing entry)  
      - True branch: proceed to AI scoring  
      - False branch: loops back to "Process Each Company" (skip duplicates)  
    - Inputs: From "Check If Company Exists"  
    - Outputs: To "AI Company Scoring" or back to "Process Each Company"  
    - Edge Cases: False negatives if Google Sheets data is stale.

---

#### 2.3 AI Scoring and CRM Integration

- **Overview:**  
  This block uses OpenAI to score companies on their likelihood to be interested in the user’s product/service, waits 2 seconds to respect Google Sheets API limits, and appends the scored company data to the CRM.

- **Nodes Involved:**  
  - AI Company Scoring  
  - Wait 2s  
  - Add Company to CRM  
  - Process Each Company (loop back)

- **Node Details:**

  - **AI Company Scoring**  
    - Type: OpenAI (LangChain)  
    - Role: Sends company data and user-defined criteria to OpenAI GPT-4.1 model for scoring.  
    - Configuration:  
      - Model: GPT-4.1  
      - Temperature: 0.2 (low randomness)  
      - System prompt: Detailed instructions including positive and negative indicators from "Set Variables"  
      - User prompt: Company details from "Filter Valid Companies" node  
      - Output: JSON with a numeric score (0-10)  
    - Inputs: From "Is New Company?" (true branch)  
    - Outputs: Score JSON to "Wait 2s"  
    - Edge Cases: API rate limits, malformed responses, prompt misconfiguration.

  - **Wait 2s**  
    - Type: Wait  
    - Role: Pauses workflow for 2 seconds to respect Google Sheets API rate limits.  
    - Configuration: Fixed 2-second wait.  
    - Inputs: From "AI Company Scoring"  
    - Outputs: To "Add Company to CRM"  
    - Edge Cases: None.

  - **Add Company to CRM**  
    - Type: Google Sheets (Append)  
    - Role: Adds the company data with AI score to the "Companies" sheet in Google Sheets CRM.  
    - Configuration:  
      - Document and sheet specified by ID and gid  
      - Columns mapped: ID, Name, Score, State ("Qualified"), Summary, Website, LinkedIn URL  
      - Append operation (no overwrite)  
    - Inputs: From "Wait 2s"  
    - Outputs: Loops back to "Process Each Company" to process next company  
    - Edge Cases: Google Sheets API errors, data type mismatches.

---

### 3. Summary Table

| Node Name           | Node Type                | Functional Role                                  | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                                                                                                                                                                                                                                                                                                                                |
|---------------------|--------------------------|-------------------------------------------------|-----------------------------|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Start               | Manual Trigger           | Entry point to start the workflow manually      | -                           | Set Variables              |                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Set Variables       | Set                      | Defines search criteria and AI scoring variables| Start                       | Search Companies           |                                                                                                                                                                                                                                                                                                                                                                                                                            |
| Search Companies    | HTTP Request             | Searches LinkedIn companies via Ghost Genius API| Set Variables               | Extract Company Data       | ## LinkedIn Company Search<br>This section initiates the workflow and searches for your target companies on LinkedIn using the Ghost Genius API.<br><br>You can filter and refine your search using keywords, company size, location, industry, or even whether the company has active job postings. Use the "Set Variables" node for it (this node also allows you to customize the AI Lead Scoring prompt).<br><br>Note that you can retrieve a maximum of 1000 companies per search (corresponding to 100 LinkedIn pages), so it's important not to exceed this number of results to avoid losing prospects.<br><br>Example: Let's say I want to target Growth Marketing Agencies with 11-50 employees. I do my search and see that there are 10,000 results. So I refine my search by using location to go country by country and retrieve all 10,000 results in several batches ranging from 500 to 1000 depending on the country.<br><br>Tips: To test the workflow or to see the number of results of your search, change the pagination parameter (Max Pages) in the "Search Companies" node. It will be displayed at the very top of the response JSON. |
| Extract Company Data | Split Out                | Splits company list into individual items       | Search Companies            | Process Each Company       | ## Company Data Processing <br>This section processes each company individually.<br><br>We retrieve all the company information using Get Company Details by using the LinkedIn link obtained from the previous section.<br><br>Then we filter the company based on the number of followers, which gives us a first indication of the company's credibility (200 in this case), and whether their LinkedIn page has a website listed.<br><br>The workflow implements batch processing with a 2-second delay between requests to respect API rate limits. This methodical approach ensures reliable data collection while preventing API timeouts.<br><br>You can adjust these thresholds based on your target market - increasing the follower count for more established businesses or decreasing it for emerging markets.<br><br>The last two modules checks if the company already exists in your database (using LinkedIn ID) to prevent duplicates because when you do close enough searches, some companies may come up several times. |
| Process Each Company | Split In Batches         | Processes companies one by one to manage rate limits | Extract Company Data, Add Company to CRM, Is New Company? (false branch) | Get Company Info, Process Each Company (false branch) | ## Company Data Processing (continued)                                                                                                                                                                                                                                                                                                                                                                                    |
| Get Company Info    | HTTP Request             | Retrieves detailed company info from API        | Process Each Company        | Filter Valid Companies     | ## Company Data Processing (continued)                                                                                                                                                                                                                                                                                                                                                                                    |
| Filter Valid Companies | If                     | Filters companies by website presence and followers count | Get Company Info            | Check If Company Exists, Process Each Company (false branch) | ## Company Data Processing (continued)                                                                                                                                                                                                                                                                                                                                                                                    |
| Check If Company Exists | Google Sheets          | Checks if company already exists in CRM          | Filter Valid Companies      | Is New Company?            | ## Company Data Processing (continued)                                                                                                                                                                                                                                                                                                                                                                                    |
| Is New Company?     | If                       | Determines if company is new (not in CRM)        | Check If Company Exists     | AI Company Scoring, Process Each Company (false branch) | ## Company Data Processing (continued)                                                                                                                                                                                                                                                                                                                                                                                    |
| AI Company Scoring  | OpenAI (LangChain)       | Scores company fit using AI based on criteria    | Is New Company?             | Wait 2s                   | ## AI Scoring and Storage<br>This section scores the company and stores it in a Google Sheet.<br><br>It's important to properly fill in the "Set variables" node at the beginning of the workflow to get a result relevant to your use case. You can also manually modify the system prompt.<br><br>Regardless of the score obtained, it's very important to always store the company. Note that we add a 2-second "wait" module to respect Google Sheet's rate limits.<br><br>We add the company to the "Companies" sheet in this [Google Sheet](https://docs.google.com/spreadsheets/d/1LfhqpyjimLjyQcmWY8mUr6YtNBcifiOVLIhAJGV9jiM/edit?usp=sharing) which you can make a copy of and use.<br><br>This AI scoring functionality is extremely impressive once perfectly configured, so I recommend taking some time to test with several companies to ensure the scoring system works well for your needs! |
| Wait 2s             | Wait                     | Waits 2 seconds to respect Google Sheets API limits | AI Company Scoring          | Add Company to CRM        | ## AI Scoring and Storage (continued)                                                                                                                                                                                                                                                                                                                                                                                    |
| Add Company to CRM  | Google Sheets            | Appends scored company data to Google Sheets CRM | Wait 2s                    | Process Each Company       | ## AI Scoring and Storage (continued)                                                                                                                                                                                                                                                                                                                                                                                    |
| Sticky Note         | Sticky Note              | Workflow documentation and guidance              | -                           | -                          | ## LinkedIn Company Search<br>This section initiates the workflow and searches for your target companies on LinkedIn using the Ghost Genius API.<br><br>You can filter and refine your search using keywords, company size, location, industry, or even whether the company has active job postings. Use the "Set Variables" node for it (this node also allows you to customize the AI Lead Scoring prompt).<br><br>Note that you can retrieve a maximum of 1000 companies per search (corresponding to 100 LinkedIn pages), so it's important not to exceed this number of results to avoid losing prospects.<br><br>Example: Let's say I want to target Growth Marketing Agencies with 11-50 employees. I do my search and see that there are 10,000 results. So I refine my search by using location to go country by country and retrieve all 10,000 results in several batches ranging from 500 to 1000 depending on the country.<br><br>Tips: To test the workflow or to see the number of results of your search, change the pagination parameter (Max Pages) in the "Search Companies" node. It will be displayed at the very top of the response JSON. |
| Sticky Note1        | Sticky Note              | Guidance on company data processing               | -                           | -                          | ## Company Data Processing <br>This section processes each company individually.<br><br>We retrieve all the company information using Get Company Details by using the LinkedIn link obtained from the previous section.<br><br>Then we filter the company based on the number of followers, which gives us a first indication of the company's credibility (200 in this case), and whether their LinkedIn page has a website listed.<br><br>The workflow implements batch processing with a 2-second delay between requests to respect API rate limits. This methodical approach ensures reliable data collection while preventing API timeouts.<br><br>You can adjust these thresholds based on your target market - increasing the follower count for more established businesses or decreasing it for emerging markets.<br><br>The last two modules checks if the company already exists in your database (using LinkedIn ID) to prevent duplicates because when you do close enough searches, some companies may come up several times. |
| Sticky Note2        | Sticky Note              | Guidance on AI scoring and storage                 | -                           | -                          | ## AI Scoring and Storage<br>This section scores the company and stores it in a Google Sheet.<br><br>It's important to properly fill in the "Set variables" node at the beginning of the workflow to get a result relevant to your use case. You can also manually modify the system prompt.<br><br>Regardless of the score obtained, it's very important to always store the company. Note that we add a 2-second "wait" module to respect Google Sheet's rate limits.<br><br>We add the company to the "Companies" sheet in this [Google Sheet](https://docs.google.com/spreadsheets/d/1LfhqpyjimLjyQcmWY8mUr6YtNBcifiOVLIhAJGV9jiM/edit?usp=sharing) which you can make a copy of and use.<br><br>This AI scoring functionality is extremely impressive once perfectly configured, so I recommend taking some time to test with several companies to ensure the scoring system works well for your needs! |
| Sticky Note4        | Sticky Note              | Introduction and overview of the template          | -                           | -                          | ## Introduction<br>Welcome to my template! Before explaining how to set it up, here's some important information:<br><br>This automation is an alternative version of [this template](https://n8n.io/workflows/3717-search-linkedin-companies-and-add-them-to-airtable-crm/) that differs by using Google Sheets instead of Airtable and adding a Lead Scoring system allowing for more precision in our targeting.<br><br>This automation therefore allows you, starting from a LinkedIn search, to enrich company data and score them to determine if they might be interested in your services/product.<br><br>For any questions, you can send me a DM on my [LinkedIn](https://www.linkedin.com/in/matthieu-belin83/) :)  |
| Sticky Note5        | Sticky Note              | Setup instructions and links                       | -                           | -                          | ## Setup<br>- Create an account on [Ghost Genius API](ghostgenius.fr) and get your API key.<br><br>- Configure the Search Companies and Get Company Info modules by creating a "Header Auth" credential:<br>**Name: "Authorization"**<br>**Value: "Bearer api_key"**<br><br>- Create a copy of this [Google Sheet](https://docs.google.com/spreadsheets/d/1LfhqpyjimLjyQcmWY8mUr6YtNBcifiOVLIhAJGV9jiM/edit?usp=sharing) by clicking on File => Make a copy (in Google Sheet).<br><br>- Configure your Google Sheet credential by following the n8n documentation.<br><br>- Create an OpenAI key [here](https://platform.openai.com/docs/overview) and add the credential to the "AI Company Scoring" node following the n8n documentation.<br><br>- Add your information to the "Set Variables" node. |
| Sticky Note6        | Sticky Note              | Tools and API references                           | -                           | -                          | ## Tools <br>**(You can use the API and CRM of your choice; this is only a suggestion)**<br><br>- API Linkedin : [Ghost Genius API](https://ghostgenius.fr) <br><br>- API Documentation : [Documentation](https://ghostgenius.fr/docs)<br>- CRM : [Google Sheet](https://workspace.google.com/intl/en/products/sheets/)<br>- AI : [OpenAI](https://openai.com)<br>- LinkedIn Location ID Finder : [Ghost Genius Locations ID Finder](https://ghostgenius.fr/tools/search-sales-navigator-locations-id) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Name: "Start"  
   - Purpose: Manual initiation of the workflow.

2. **Create Set Node**  
   - Name: "Set Variables"  
   - Purpose: Define search and scoring parameters.  
   - Assign the following string variables:  
     - "Your target": e.g., "Growth Marketing Agency"  
     - Company size code (e.g., "C")  
     - Location ID (numeric, from Ghost Genius location tool)  
     - "Your product or service": e.g., "our CRM implementation services"  
     - "Positive indicators": multiline string describing factors indicating good fit  
     - "Negative indicators": multiline string describing factors indicating poor fit  
   - Connect "Start" → "Set Variables".

3. **Create HTTP Request Node for Company Search**  
   - Name: "Search Companies"  
   - URL: `https://api.ghostgenius.fr/v2/search/companies`  
   - Authentication: HTTP Header Auth with Bearer token (Ghost Genius API key)  
   - Query Parameters:  
     - keywords = `={{ $json['Your target'] }}`  
     - locations = `={{ $json['Location (find it on : https://www.ghostgenius.fr/tools/search-sales-navigator-locations-id)'] }}`  
     - company_size = `={{ $json['B: 1-10 employees, C: 11-50 employees, D: 51-200 employees, E: 201-500 employees, F: 501-1000 employees, G: 1001-5000 employees, H: 5001-10,000 employees, I: 10,001+ employees'] }}`  
   - Pagination: Enable with max 3 pages, 2-second interval between requests  
   - Connect "Set Variables" → "Search Companies".

4. **Create Split Out Node**  
   - Name: "Extract Company Data"  
   - Field to split out: "data" (array of companies from API response)  
   - Connect "Search Companies" → "Extract Company Data".

5. **Create Split In Batches Node**  
   - Name: "Process Each Company"  
   - Batch size: 1 (default)  
   - Connect "Extract Company Data" → "Process Each Company".

6. **Create HTTP Request Node for Company Details**  
   - Name: "Get Company Info"  
   - URL: `https://api.ghostgenius.fr/v2/company`  
   - Authentication: HTTP Header Auth with Bearer token  
   - Query Parameter: "url" = `={{ $json.url }}` (LinkedIn URL of company)  
   - Enable batching with 1 request per 2 seconds to respect rate limits  
   - Enable retry on failure  
   - Connect "Process Each Company" (main output) → "Get Company Info".

7. **Create If Node to Filter Valid Companies**  
   - Name: "Filter Valid Companies"  
   - Conditions:  
     - website is not empty  
     - followers_count > 200  
   - True output → "Check If Company Exists"  
   - False output → loops back to "Process Each Company" (to skip invalid companies).

8. **Create Google Sheets Node for Lookup**  
   - Name: "Check If Company Exists"  
   - Operation: Lookup  
   - Sheet: "Companies" in your Google Sheet CRM  
   - Lookup column: "ID"  
   - Lookup value: `={{ $json.id }}`  
   - Credentials: Google Sheets OAuth2  
   - Always output data  
   - Connect "Filter Valid Companies" (true branch) → "Check If Company Exists".

9. **Create If Node to Check New Company**  
   - Name: "Is New Company?"  
   - Condition: Check if lookup result is empty (meaning company does not exist)  
   - True output → "AI Company Scoring"  
   - False output → loops back to "Process Each Company" (skip duplicates)  
   - Connect "Check If Company Exists" → "Is New Company?".

10. **Create OpenAI Node for AI Scoring**  
    - Name: "AI Company Scoring"  
    - Model: GPT-4.1  
    - Temperature: 0.2  
    - System Prompt: Include instructions to score company from 0 to 10 based on:  
      - Industry fit  
      - Company profile  
      - Pain points  
      - Positive and negative indicators from "Set Variables"  
    - User Prompt: Include company details from "Filter Valid Companies" node (name, description, employees, industry, specialties, location, founding date, funding)  
    - Output: JSON with numeric "score"  
    - Credentials: OpenAI API key  
    - Connect "Is New Company?" (true branch) → "AI Company Scoring".

11. **Create Wait Node**  
    - Name: "Wait 2s"  
    - Duration: 2 seconds  
    - Purpose: To respect Google Sheets API rate limits  
    - Connect "AI Company Scoring" → "Wait 2s".

12. **Create Google Sheets Node to Append Data**  
    - Name: "Add Company to CRM"  
    - Operation: Append row to "Companies" sheet in your Google Sheet CRM  
    - Columns mapped:  
      - ID: `={{ $('Get Company Info').item.json.id }}`  
      - Name: `={{ $('Get Company Info').item.json.name }}`  
      - Score: `={{ $json.message.content.score }}` (from AI scoring)  
      - State: "Qualified" (static)  
      - Summary: `={{ $('Get Company Info').item.json.description }}`  
      - Website: `={{ $('Get Company Info').item.json.website }}`  
      - LinkedIn: `={{ $('Get Company Info').item.json.url }}`  
    - Credentials: Google Sheets OAuth2  
    - Connect "Wait 2s" → "Add Company to CRM".

13. **Loop Back**  
    - Connect "Add Company to CRM" → "Process Each Company" (to process next company batch).

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| This automation is an alternative version of [this template](https://n8n.io/workflows/3717-search-linkedin-companies-and-add-them-to-airtable-crm/) that uses Google Sheets instead of Airtable and adds a Lead Scoring system for more precise targeting. For questions, contact the author on [LinkedIn](https://www.linkedin.com/in/matthieu-belin83/).                                                                                                                                                                   | Template origin and author contact                                                                       |
| Setup instructions: Create Ghost Genius API account and get API key; configure HTTP Header Auth credentials; make a copy of the provided Google Sheet template; configure Google Sheets and OpenAI credentials per n8n documentation; customize "Set Variables" node.                                                                                                                                                                                                                                         | Setup instructions                                                                                       |
| Tools used: Ghost Genius API for LinkedIn data, Google Sheets as CRM, OpenAI for AI scoring. Location IDs for LinkedIn searches can be found using [Ghost Genius Locations ID Finder](https://ghostgenius.fr/tools/search-sales-navigator-locations-id).                                                                                                                                                                                                                                                         | Tools and API references                                                                                  |
| The workflow respects API rate limits by batching requests and adding delays (2 seconds between requests for company info and before Google Sheets append). Adjust thresholds (e.g., follower count) and pagination limits to fit your target market and API constraints.                                                                                                                                                                                                                                      | Rate limiting and performance notes                                                                      |
| AI scoring prompt is customizable in the "AI Company Scoring" node. Positive and negative indicators should be carefully tailored to your product/service for best results. Always store companies regardless of score for full tracking.                                                                                                                                                                                                                                                                    | AI scoring customization advice                                                                           |
| Google Sheet template used: [Google Sheet CRM](https://docs.google.com/spreadsheets/d/1LfhqpyjimLjyQcmWY8mUr6YtNBcifiOVLIhAJGV9jiM/edit?usp=sharing). Make a personal copy before use.                                                                                                                                                                                                                                                                                                                          | Google Sheet CRM template                                                                                 |

---

This documentation provides a comprehensive understanding and step-by-step reproduction guide for the workflow "Search LinkedIn companies, Score with AI and add them to Google Sheet CRM." It covers all nodes, their configurations, interconnections, and potential edge cases to facilitate maintenance, customization, and troubleshooting.