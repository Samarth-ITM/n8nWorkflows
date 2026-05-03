Automated Website Audit & Personalized Outreach with Lighthouse and GPT-4

https://n8nworkflows.xyz/workflows/automated-website-audit---personalized-outreach-with-lighthouse-and-gpt-4-5940


# Automated Website Audit & Personalized Outreach with Lighthouse and GPT-4

### 1. Workflow Overview

This workflow automates a complete process for website auditing and personalized outreach by integrating Google Sheets, web scraping, Lighthouse performance audits, and AI-powered content generation using GPT-4. It is designed for businesses or agencies aiming to analyze potential clients' websites and send hyper-personalized outreach emails based on detailed website and business insights.

The workflow is logically divided into four main blocks:

- **1.1 Trigger & CRM Input**: Receives input data (emails, URLs) from Google Sheets and initiates processing.
- **1.2 Scraping Business Data**: Crawls and scrapes business profile pages to extract detailed company info including contact data, staff, and business description.
- **1.3 Analyzing Lighthouse Stats + Website UI/UX Design & Generating Outreach Email**: Performs a Google Lighthouse audit on the scraped or constructed website URL, captures screenshots, analyzes performance data, and uses AI to generate tailored outreach messages.
- **1.4 Analyzing Website Errors & Generating Outreach Email**: Handles cases where the website is not accessible or returns errors, generating specialized outreach messages for these error scenarios.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & CRM Input

**Overview:**  
This block triggers the workflow from a webhook or scheduled wait nodes, fetches new rows from a Google Sheet, and verifies email presence to decide further processing.

**Nodes Involved:**  
- Webhook  
- Get New Rows (Google Sheets)  
- FirstRow (Code)  
- If Email is not Empty (IF node)  
- If Email is not BokaDirect's support email (IF node)  
- Update Sheet (email doesn't exist) (Google Sheets)  
- Wait, Wait2 (Wait nodes)

**Node Details:**

- **Webhook**  
  - Type: Trigger node  
  - Purpose: Entry point for the workflow via HTTP request.  
  - Config: Custom path configured.  
  - Connections: Outputs to "Get New Rows".  
  - Edge cases: Incorrect or missing webhook calls may stall workflow start.

- **Get New Rows**  
  - Type: Google Sheets node (read operation)  
  - Purpose: Reads new CRM entries (e.g., leads/emails) from the Google Sheet.  
  - Config: Document ID and sheet name parameters (not shown, user must configure).  
  - Credentials: Google Sheets OAuth2.  
  - OnError: Continues on error to prevent total failure.  
  - Connections: Outputs to "FirstRow" and a Wait node.  
  - Edge cases: API limits, auth expiration, empty sheets.

- **FirstRow**  
  - Type: Code node  
  - Purpose: Takes only the first row from the fetched data for processing.  
  - Code: `return [items[0]];`  
  - Connections: Outputs to "If Email is not Empty".  
  - Edge cases: Empty input array will cause errors.

- **If Email is not Empty**  
  - Type: IF node  
  - Purpose: Checks if the email field from the first row is present and not empty.  
  - Condition: Checks `$json.Email` is not empty string.  
  - Outputs: True branch continues processing; False branch updates sheet and loops back.  
  - Edge cases: Missing or malformed "Email" field.

- **If Email is not BokaDirect's support email**  
  - Type: IF node  
  - Purpose: Filters out a specific internal or known support email from processing.  
  - Condition: `$json.Email` not equal to "foto@lkjkjkj.com".  
  - Outputs: True branch continues; False branch updates sheet and loops.  
  - Edge cases: Hardcoded email may need updating for different deployments.

- **Update Sheet (email doesn't exist)**  
  - Type: Google Sheets node (update operation)  
  - Purpose: Updates Google Sheet when email is missing or filtered out.  
  - Config: Requires sheet name and document ID.  
  - Credentials: Google Sheets OAuth2.  
  - OnError: Continue on error.  
  - Connections: Loops back to "Get New Rows" (iteration).  
  - Edge cases: Write permission issues.

- **Wait nodes (Wait, Wait2)**  
  - Purpose: Control pacing of workflow execution to avoid API rate limits or timing issues.  
  - Config: Default or specified delays.  
  - Edge cases: If waits are too short, APIs can throttle; too long increases total workflow time.

---

#### 1.2 Scraping Business Data

**Overview:**  
This block requests the business profile URL, analyzes the returned HTML for presence of key elements, extracts emails, business details, staff lists, and contact information through custom code.

**Nodes Involved:**  
- BokaDirect Profile URL Request (HTTP Request)  
- ProfilePage or Redirected (Code)  
- Get Email (Code)  
- If ProfilePage or not (IF node)  
- Business Details (Code)  
- Update Sheet URL redirected (Google Sheets)  
- Update Sheet URL redirected (Google Sheets)  
- Wait3 (Wait node)

**Node Details:**

- **BokaDirect Profile URL Request**  
  - Type: HTTP Request node  
  - Purpose: Fetches the HTML content of the business profile URL provided in input.  
  - Config: URL taken from `$json['BokaDirect Profile']`.  
  - Edge cases: HTTP errors, redirects, timeouts.

- **ProfilePage or Redirected**  
  - Type: Code node  
  - Purpose: Checks if the fetched page contains a specific HTML `<div class="w-full">` to determine if it’s the profile page or a redirect.  
  - Code: Uses regex test on HTML content.  
  - Output: Boolean flag `hasWFullDiv`.  
  - Edge cases: HTML structure changes may break detection.

- **Get Email**  
  - Type: Code node  
  - Purpose: Extracts the first email address found in the HTML content using regex.  
  - Output: `emailFound` boolean and `email` string.  
  - Edge cases: HTML encoding or obfuscation may prevent email extraction.

- **If ProfilePage or not**  
  - Type: IF node  
  - Purpose: Branches based on whether the page is profile page or redirect.  
  - Condition: `hasWFullDiv === true`  
  - True branch: proceeds to extract business details.  
  - False branch: updates sheet to mark URL redirected.  
  - Edge cases: Incorrect boolean evaluation.

- **Business Details**  
  - Type: Code node  
  - Purpose: Parses detailed info from HTML including business name, description, staff (names and roles), URL, phone, location, and attempts to decode email from base64 embedded image.  
  - Logic: Uses multiple regex patterns, HTML entity decoding, base64 decoding.  
  - Output: Structured JSON with business details and staff list.  
  - Edge cases: HTML changes, missing data, base64 decoding failures.

- **Update Sheet URL redirected**  
  - Type: Google Sheets node (update)  
  - Purpose: Updates the Google Sheet when the profile URL redirects or does not contain expected data.  
  - OnError: Continue on error.  
  - Edge cases: Sheet update permissions.

- **Wait3**  
  - Purpose: Introduces a delay after sheet update.  
  - Edge cases: As above.

---

#### 1.3 Analyzing Lighthouse Stats + Website UI/UX Design & Generating Outreach Email

**Overview:**  
This block performs a Google Lighthouse audit on the website, captures a screenshot, analyzes the audit results and site design, and uses AI to craft a personalized outreach email based on these insights.

**Nodes Involved:**  
- DecisionMaker Name (OpenAI GPT-4 node)  
- If website exist (IF node)  
- Professional Email or Personal (Code)  
- Professional or Not (IF node)  
- Set URL (Set node)  
- Set URL from Professional Email (Set node)  
- Update Scraped Business Details (Google Sheets)  
- Business Website Request (HTTP Request)  
- Lighthouse Stats Request (HTTP Request)  
- Filter Stats (Code)  
- Screenshot of Website Request (HTTP Request)  
- Analyse UI/UX design, Lighthouse stats & Business details (OpenAI GPT-4 node)  
- Update Sheet with messages and Status (Google Sheets)  
- Wait1, Wait4, Wait5, Wait6, Wait8 (Wait nodes)

**Node Details:**

- **DecisionMaker Name**  
  - Type: OpenAI GPT-4 node (LangChain)  
  - Purpose: Uses staff list and email data to identify the decision maker's name from roles.  
  - Model: GPT-4o-mini.  
  - Input: Staff list and email from previous nodes.  
  - Output: JSON with `DecisionMakerName`.  
  - Edge cases: Ambiguous staff roles, missing data.

- **If website exist**  
  - Type: IF node  
  - Purpose: Checks if the extracted business URL is valid and not an Instagram page or empty.  
  - Conditions include URL not starting with "/", not "none", not containing "instagram", and not empty.  
  - Branches: True goes to use extracted URL; False proceeds to check email domain.  
  - Edge cases: Edge case URLs, false positives.

- **Professional Email or Personal**  
  - Type: Code node  
  - Purpose: Determines if the email is from a professional or free/personal provider (gmail, yahoo, etc.).  
  - Output: `isProfessional` boolean, `emailType` string, and domain if professional.  
  - Edge cases: New or uncommon email domains.

- **Professional or Not**  
  - Type: IF node  
  - Purpose: Branches based on whether email is professional.  
  - True: Sets URL from professional email domain.  
  - False: Updates sheet that website doesn't exist.  
  - Edge cases: Email parsing errors.

- **Set URL / Set URL from Professional Email**  
  - Type: Set node  
  - Purpose: Assigns the URL field for further requests, either from business details or constructed from professional email domain.  
  - Edge cases: URL formatting issues.

- **Update Scraped Business Details**  
  - Type: Google Sheets node (update)  
  - Purpose: Updates sheet with the determined URL and scraped business details.  
  - Edge cases: Write permission errors.

- **Business Website Request**  
  - Type: HTTP Request node  
  - Purpose: Requests the business website using the determined URL for Lighthouse auditing.  
  - OnError: Continue on error to handle failures gracefully.  
  - Edge cases: Site unreachable, timeouts.

- **Lighthouse Stats Request**  
  - Type: HTTP Request node  
  - Purpose: Calls Google PageSpeed Insights API to get Lighthouse audit data for categories: performance, accessibility, SEO, best practices.  
  - Config: URL parameterized with the business URL; API key required.  
  - Edge cases: API limits, invalid keys, request failures.

- **Filter Stats**  
  - Type: Code node  
  - Purpose: Extracts and rounds Lighthouse category scores from API response.  
  - Output: JSON object with scores or "none".  
  - Edge cases: Missing or malformed API response.

- **Screenshot of Website Request**  
  - Type: HTTP Request node  
  - Purpose: Fetches a full-page screenshot from an external service (thum.io) based on the business URL.  
  - Edge cases: Service limits, request failures.

- **Analyse UI/UX design, Lighthouse stats & Business details**  
  - Type: OpenAI GPT-4 node (LangChain)  
  - Purpose: Performs AI analysis combining screenshot, Lighthouse scores, and business info to generate a personalized cold outreach email icebreaker.  
  - Input: Base64 screenshot + text data.  
  - Output: JSON with `icebreaker` message.  
  - Edge cases: Model response errors, prompt misinterpretation.

- **Update Sheet with messages and Status**  
  - Type: Google Sheets node (update)  
  - Purpose: Writes generated AI outreach messages and status back to Google Sheets.  
  - Edge cases: Sheet write issues.

- **Wait nodes**  
  - Purpose: Manage pacing and API rate limits post various update steps.

---

#### 1.4 Analyzing Website Error & Generating Outreach Email

**Overview:**  
When the website request fails (e.g., 404, timeout, SSL errors), this block generates a specialized outreach message explaining the error and offering web repair services.

**Nodes Involved:**  
- Write message for error in Website (OpenAI GPT-4 node)  
- Update Error in Stats (Google Sheets)  
- Update Sheet (Google Sheets)  
- Wait7 (Wait node)

**Node Details:**

- **Write message for error in Website**  
  - Type: OpenAI GPT-4 node (LangChain)  
  - Purpose: Generates a personalized cold outreach email icebreaker addressing the specific website error found, emphasizing potential business impact and offering help.  
  - Input: Business details, decision maker name, and error details from website request node.  
  - Output: JSON with personalized message content.  
  - Edge cases: Model generation failure, incomplete error info.

- **Update Error in Stats**  
  - Type: Google Sheets node (update)  
  - Purpose: Updates the Google Sheet with error info and generated message.  
  - Edge cases: Write errors.

- **Update Sheet**  
  - Type: Google Sheets node (update)  
  - Purpose: Final sheet update to record the outreach message and status.  
  - Edge cases: Write errors.

- **Wait7**  
  - Purpose: Delay after sheet update to pace workflow.

---

### 3. Summary Table

| Node Name                        | Node Type                     | Functional Role                                      | Input Node(s)                  | Output Node(s)                               | Sticky Note                                                                                          |
|---------------------------------|-------------------------------|----------------------------------------------------|-------------------------------|----------------------------------------------|----------------------------------------------------------------------------------------------------|
| Webhook                         | Webhook                       | Entry point trigger                                | -                             | Get New Rows                                 | # Step 1. Trigger & CRM Input                                                                      |
| Get New Rows                    | Google Sheets                 | Fetch new CRM rows                                | Webhook, Wait                  | FirstRow, Wait                               | # Step 1. Trigger & CRM Input                                                                      |
| FirstRow                       | Code                         | Select first row for processing                    | Get New Rows                  | If Email is not Empty                        | # Step 1. Trigger & CRM Input                                                                      |
| If Email is not Empty           | IF                            | Check for presence of email                         | FirstRow                      | If Email is not BokaDirect's support email, Update Sheet (email doesn't exist) | # Step 1. Trigger & CRM Input                                                                      |
| If Email is not BokaDirect's support email | IF                     | Filter out known support email                      | If Email is not Empty         | BokaDirect Profile URL Request, Update Sheet (email doesn't exist) | # Step 1. Trigger & CRM Input                                                                      |
| Update Sheet (email doesn't exist) | Google Sheets             | Update sheet for missing or excluded email         | If Email is not Empty, If Email is not BokaDirect's support email | Get New Rows, Wait2                   | # Step 1. Trigger & CRM Input                                                                      |
| Wait                           | Wait                         | Controls pacing before next sheet fetch             | Get New Rows                  | Get New Rows                                 | # Step 1. Trigger & CRM Input                                                                      |
| Wait2                          | Wait                         | Controls pacing after sheet update                  | Update Sheet (email doesn't exist) | Update Sheet (email doesn't exist)         | # Step 1. Trigger & CRM Input                                                                      |
| BokaDirect Profile URL Request  | HTTP Request                 | Request business profile page HTML                   | If Email is not BokaDirect's support email | ProfilePage or Redirected               | # Step 2. Scraping Business Data                                                                   |
| ProfilePage or Redirected       | Code                         | Determine if page is profile or redirect             | BokaDirect Profile URL Request | Get Email, Update Sheet URL redirected       | # Step 2. Scraping Business Data                                                                   |
| Get Email                      | Code                         | Extract email from HTML content                       | ProfilePage or Redirected      | If ProfilePage or not                        | # Step 2. Scraping Business Data                                                                   |
| If ProfilePage or not           | IF                            | Branch on profile page presence                       | Get Email                     | Business Details, Update Sheet URL redirected | # Step 2. Scraping Business Data                                                                   |
| Business Details               | Code                         | Parse business info, staff, contact details          | If ProfilePage or not          | DecisionMaker Name                           | # Step 2. Scraping Business Data                                                                   |
| DecisionMaker Name             | OpenAI GPT-4 (LangChain)      | AI to identify decision maker from staff list        | Business Details              | If website exist                            | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| If website exist               | IF                            | Check if business URL is valid and usable             | DecisionMaker Name            | Set URL, Professional Email or Personal      | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Professional Email or Personal | Code                         | Determine if email domain is professional or personal | If website exist              | Professional or Not                          | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Professional or Not            | IF                            | Branch on email professional status                   | Professional Email or Personal | Set URL from Professional Email, Update Website doesn't exist | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Set URL                       | Set                          | Set URL from business details                          | If website exist              | Update Scraped Business Details             | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Set URL from Professional Email | Set                          | Construct URL from professional email domain          | Professional or Not           | Update Scraped Business Details             | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Update Website doesn't exist  | Google Sheets                 | Update sheet when no website exists                    | Professional or Not           | Wait4                                        | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Update Scraped Business Details | Google Sheets                 | Update sheet with scraped business details and URL    | Set URL, Set URL from Professional Email | Business Website Request, Wait5        | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Business Website Request      | HTTP Request                 | Request business website HTML                           | Update Scraped Business Details | Lighthouse Stats Request, Write message for error in Website | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Lighthouse Stats Request      | HTTP Request                 | Request Lighthouse audit data via Google API           | Business Website Request      | Filter Stats, Update Error in Stats          | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Filter Stats                 | Code                         | Extract and round Lighthouse category scores           | Lighthouse Stats Request      | Screenshot of Website Request                | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Screenshot of Website Request | HTTP Request                 | Get full-page screenshot from external service         | Filter Stats                 | Analyse UI/UX design, Lighthouse stats & Business details, Wait1 | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Analyse UI/UX design, Lighthouse stats & Business details | OpenAI GPT-4 (LangChain) | Generate personalized outreach email copy based on audit and business data | Screenshot of Website Request | Update Sheet with messages and Status   | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Update Sheet with messages and Status | Google Sheets           | Update sheet with generated outreach message and status | Analyse UI/UX design...      | Wait8                                        | # Step 3. Analyzing Lighthouse Stats + Website UI/UX Design & Generating Hyper-Personalised Outreach Email |
| Write message for error in Website | OpenAI GPT-4 (LangChain)  | Generate outreach email when website request returns error | Business Website Request     | Update Sheet                                 | # Step 4. Analyzing Website Error & Generating Hyper-Personalised Outreach Email                    |
| Update Error in Stats        | Google Sheets                 | Update error details and message in sheet               | Lighthouse Stats Request      | Wait7                                        | # Step 4. Analyzing Website Error & Generating Hyper-Personalised Outreach Email                    |
| Update Sheet                 | Google Sheets                 | Final update with error outreach message                 | Write message for error in Website | Wait6                                      | # Step 4. Analyzing Website Error & Generating Hyper-Personalised Outreach Email                    |
| Wait1, Wait3, Wait4, Wait5, Wait6, Wait7, Wait8 | Wait             | Pacing and delay nodes between processing steps         | Various                      | Various                                      | Various pacing nodes across the workflow                                                          |
| Sticky Note, Sticky Note1, Sticky Note2, Sticky Note3 | Sticky Note           | Visual labels grouping workflow steps                     | -                           | -                                            | Step 1 to Step 4 labeled for clarity                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node**  
   - Type: Webhook  
   - Path: Unique identifier (e.g., "0f267a89-60f6-46c5-a97e-39c4e1452579")  
   - Purpose: Entry trigger.

2. **Create Google Sheets Node ("Get New Rows")**  
   - Operation: Read rows  
   - Setup credentials for Google Sheets OAuth2  
   - Configure Document ID and Sheet Name where leads/emails are stored.

3. **Create Code Node ("FirstRow")**  
   - Code: `return [items[0]];`  
   - Purpose: Process only first row for performance.

4. **Create IF Node ("If Email is not Empty")**  
   - Condition: Check `$json.Email` is not empty string.  
   - True branch continues; False branch to update sheet for missing email.

5. **Create IF Node ("If Email is not BokaDirect's support email")**  
   - Condition: `$json.Email` not equals `"foto@lkjkjkj.com"`.  
   - True branch continues; False branch updates sheet.

6. **Create Google Sheets Node ("Update Sheet (email doesn't exist)")**  
   - Operation: Update  
   - Configure document and sheet.  
   - Purpose: Mark emails missing or filtered out.

7. **Create Wait Nodes ("Wait", "Wait2")**  
   - Use default or configured delay.

8. **Connect Webhook → Get New Rows → FirstRow → If Email is not Empty → If Email is not BokaDirect's support email → BokaDirect Profile URL Request**

9. **Create HTTP Request ("BokaDirect Profile URL Request")**  
   - URL: `={{ $json['BokaDirect Profile'] }}`

10. **Create Code Node ("ProfilePage or Redirected")**  
    - Check for `<div class="w-full">` in fetched HTML to decide profile page or redirect.

11. **Create Code Node ("Get Email")**  
    - Regex extract first email from HTML.

12. **Create IF Node ("If ProfilePage or not")**  
    - Condition: `hasWFullDiv === true`  
    - True: to "Business Details"  
    - False: to "Update Sheet URL redirected"

13. **Create Code Node ("Business Details")**  
    - Complex regex and decoding for business details, staff, contact info, and email from base64 image.

14. **Create Google Sheets Node ("Update Sheet URL redirected")**  
    - Update operation for redirect cases.

15. **Create OpenAI GPT-4 Node ("DecisionMaker Name")**  
    - Model: GPT-4o-mini  
    - Input: Staff list and email  
    - Output: Decision maker name JSON

16. **Create IF Node ("If website exist")**  
    - Conditions on business URL validity.

17. **Create Code Node ("Professional Email or Personal")**  
    - Determine email type based on domain.

18. **Create IF Node ("Professional or Not")**  
    - Branch based on email type.

19. **Create Set Nodes ("Set URL", "Set URL from Professional Email")**  
    - Assign URL for further requests.

20. **Create Google Sheets Node ("Update Scraped Business Details")**  
    - Update business URL and details.

21. **Create HTTP Request ("Business Website Request")**  
    - Request business website with URL.

22. **Create HTTP Request ("Lighthouse Stats Request")**  
    - Call Google PageSpeed Insights API with URL and API key.

23. **Create Code Node ("Filter Stats")**  
    - Extract and round Lighthouse scores.

24. **Create HTTP Request ("Screenshot of Website Request")**  
    - Request full-page screenshot from thum.io.

25. **Create OpenAI GPT-4 Node ("Analyse UI/UX design, Lighthouse stats & Business details")**  
    - Input: Screenshot (base64), stats, business info  
    - Output: Personalized icebreaker email copy.

26. **Create Google Sheets Node ("Update Sheet with messages and Status")**  
    - Update sheet with generated message and status.

27. **Create OpenAI GPT-4 Node ("Write message for error in Website")**  
    - Input: Business info, error details  
    - Output: Personalized error outreach message.

28. **Create Google Sheets Nodes ("Update Error in Stats", "Update Sheet")**  
    - Update error messages and general sheet update for error cases.

29. **Add Wait Nodes ("Wait1" to "Wait8") between steps as required for pacing and API limits.**

30. **Connect all nodes following the logical flow and branches as outlined in the summary table.**

31. **Set all credentials properly: Google Sheets OAuth2, OpenAI API, any external API keys (Google PageSpeed API, thum.io).**

32. **Test each branch, especially error handling and edge cases (missing emails, redirects, website errors).**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                       | Context or Link                                    |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Google PageSpeed Insights API requires a valid API key for Lighthouse audit requests; replace `**YourPassKey**` with a real key.                                                                                                                                  | https://developers.google.com/speed/pagespeed/insights/api |
| The workflow uses external screenshot service `thum.io` for website screenshots; ensure API access and quota.                                                                                                                                                     | https://www.thum.io/                              |
| Email extraction from base64-encoded images may fail if encoding changes; consider updating regex or decoding logic if scraping fails.                                                                                                                            | Business Details node code                         |
| Hardcoded email filter `"foto@lkjkjkj.com"` must be updated to reflect actual internal support emails to avoid false exclusions.                                                                                                                                  | If Email is not BokaDirect's support email node  |
| AI model used is GPT-4o-mini through LangChain node integration; requires OpenAI API credentials with appropriate quota and permissions.                                                                                                                          | OpenAI GPT-4 nodes                                |
| Workflow includes multiple pacing wait nodes to avoid API rate limits and ensure sequential processing; adjust delays as needed for your environment.                                                                                                             | Wait nodes throughout workflow                     |
| The workflow’s logic and node connections are designed to loop back and process multiple rows iteratively from Google Sheets until completion.                                                                                                                  | Looping via Update Sheet nodes to Get New Rows   |

---

**Disclaimer:**  
The text provided is exclusively from a workflow automated with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data processed is legal and public.