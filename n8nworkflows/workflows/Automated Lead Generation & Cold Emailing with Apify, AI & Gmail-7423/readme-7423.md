Automated Lead Generation & Cold Emailing with Apify, AI & Gmail

https://n8nworkflows.xyz/workflows/automated-lead-generation---cold-emailing-with-apify--ai---gmail-7423


# Automated Lead Generation & Cold Emailing with Apify, AI & Gmail

### 1. Workflow Overview

This workflow automates lead generation and cold emailing by integrating form input, web data scraping, AI-powered email content creation, and email dispatch via Gmail. It is designed for digital marketing agencies or sales teams aiming to streamline outreach to potential clients based on specified business types and locations.

The workflow is logically grouped into the following functional blocks:

- **1.1 Input Reception:** Captures user input about the target business type, location, lead quantity, and email style through a web form.
- **1.2 Lead Data Acquisition:** Sends a request to an Apify actor to scrape business leads based on the form input.
- **1.3 Lead Filtering and Storage:** Filters leads with valid websites and appends them to a Google Sheet for record-keeping.
- **1.4 Email Address Extraction:** Uses Google Gemini AI to extract the best email address from each lead’s website.
- **1.5 Email Generation:** Utilizes OpenAI GPT-4 to generate personalized cold email subject lines and bodies based on lead details and selected email style.
- **1.6 Email Sending and Logging:** Sends the generated emails via Gmail, waits between sends to avoid spam detection, and updates the Google Sheet with email send status and timestamp.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  Captures form inputs specifying the lead search criteria: business type, location, number of leads, and preferred email style.

- **Nodes Involved:**  
  - On form submission  
  - Sticky Note ("# Business Data")

- **Node Details:**  
  - **On form submission**  
    - Type: `formTrigger` (Webhook trigger node)  
    - Configuration: Custom form titled "Lead Machine" with required fields: Business Type (text), Location (text), Lead Number (numeric), and Email Style (dropdown with Friendly, Professional, Simple)  
    - Input: User HTTP POST submission  
    - Output: JSON containing form values  
    - Edge Cases: Missing required fields block submission; malformed input possible if form validation fails.

  - **Sticky Note**  
    - Provides a visual label for the block; no functional role.

---

#### 2.2 Lead Data Acquisition

- **Overview:**  
  Sends an HTTP POST request to an Apify actor endpoint that scrapes leads based on input criteria, returning business data.

- **Nodes Involved:**  
  - HTTP Request

- **Node Details:**  
  - **HTTP Request**  
    - Type: HTTP Request node  
    - Configuration:  
      - URL: Dynamic, replaced with `Apify_Actor_Endpoint_URL` environment/credential variable  
      - Method: POST  
      - Body: JSON containing parameters such as location, business type (searchStringsArray), max leads, and scraping options (mostly false except `scrapeReviewsPersonalData`)  
    - Input: JSON from form submission node  
    - Output: JSON array with business leads data, including website, address, category, title, and phone number  
    - Edge Cases: Network errors, invalid Apify endpoint URL, or empty response if no leads found.

---

#### 2.3 Lead Filtering and Storage

- **Overview:**  
  Filters leads to only those containing a website URL and appends these filtered leads to a Google Sheet.

- **Nodes Involved:**  
  - Filter  
  - Append row in sheet  
  - Sticky Note ("# Getting the Email Address")

- **Node Details:**  
  - **Filter**  
    - Type: Filter node  
    - Configuration: Checks if the `website` property exists and is non-empty in the lead data.  
    - Input: Leads array from HTTP Request  
    - Output: Leads with valid websites (true branch), or discarded leads (false branch)  
    - Edge Cases: Leads without websites are ignored; expression errors if `website` field missing or malformed.

  - **Append row in sheet**  
    - Type: Google Sheets node  
    - Configuration: Appends lead details (Address, Website, Category, Company Name, Email Address (initially empty), Phone Number) to Sheet1 of a specified Google Sheet document ID.  
    - Input: Filtered lead data  
    - Output: Confirmation of append operation  
    - Credentials: Google Sheets OAuth2 configured externally  
    - Edge Cases: Sheet access permission errors, API quota limits, or malformed data mapping.

  - **Sticky Note**  
    - Labels the processing block for clarity.

---

#### 2.4 Email Address Extraction

- **Overview:**  
  For each lead with a website, uses Google Gemini AI to scrape and extract the best email address from the website content.

- **Nodes Involved:**  
  - Loop Over Items  
  - Information Extractor (Google Gemini Chat Model)  
  - If (valid email address check)

- **Node Details:**  
  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Configuration: Iterates over each lead row appended to Google Sheets  
    - Edge Cases: Large lead counts may require batch size tuning to avoid timeouts.

  - **Information Extractor**  
    - Type: LangChain Information Extractor with Google Gemini Chat Model  
    - Configuration: Extracts a single, ideally formatted "Email Address" attribute from the website field JSON.  
    - Input: Lead website string  
    - Output: JSON with extracted "Email Address"  
    - Credentials: Google Gemini API key required  
    - Edge Cases: Failure to find an email, malformed website URLs, API quota or connectivity issues.

  - **If**  
    - Type: Conditional node  
    - Configuration: Passes only leads where extracted email contains "@" symbol to next step; otherwise ends flow for that lead.  
    - Edge Cases: False positives or invalid emails may proceed; no email leads skipped.

---

#### 2.5 Email Generation

- **Overview:**  
  Generates personalized cold email subject and body using OpenAI GPT-4 based on lead details and chosen email style.

- **Nodes Involved:**  
  - Wait  
  - Information Extractor1 (OpenAI Chat Model)  
  - Edit Fields1

- **Node Details:**  
  - **Wait**  
    - Type: Wait node  
    - Configuration: 1-second delay between email generation for rate limiting and API pacing.  
    - Edge Cases: Minimal risk, but long runs may accumulate delay.

  - **Information Extractor1**  
    - Type: LangChain Information Extractor with OpenAI Chat Model (GPT-4 mini)  
    - Configuration:  
      - Input prompt includes company name, business type, and email style from form input.  
      - Instructions specify tone, greeting format, plural pronouns, professional and concise style, and signature format.  
      - Outputs required: "Mail Subject" and "Mail Body" (cold email content).  
    - Credentials: OpenAI API key required  
    - Edge Cases: Model errors, API limits, or unexpected prompt results.

  - **Edit Fields1**  
    - Type: Set node  
    - Configuration: Adds current send time (formatted) and email address to the data object for logging.  
    - Edge Cases: Time formatting errors unlikely.

---

#### 2.6 Email Sending and Logging

- **Overview:**  
  Sends the generated cold email via Gmail and updates the Google Sheet with send time and status.

- **Nodes Involved:**  
  - Send a message (Gmail)  
  - Append or update row in sheet  
  - No Operation, do nothing  
  - Sticky Notes ("# Email Send" and setup guide notes)

- **Node Details:**  
  - **Send a message**  
    - Type: Gmail node  
    - Configuration:  
      - Sends email to the extracted email address  
      - Uses subject and message body from the OpenAI-generated content  
      - Email format: plain text  
      - Retry on error: no; max 2 tries  
    - Credentials: Gmail OAuth2 configured externally  
    - Edge Cases: Gmail API limits, invalid email addresses, or send failures; node set to continue on error.

  - **Append or update row in sheet**  
    - Type: Google Sheets node  
    - Configuration: Updates the lead’s row by matching on "Email Address" with status "✅" and current "SEND Time" in the sheet.  
    - Edge Cases: Sheet update conflicts or access errors.

  - **No Operation, do nothing**  
    - Type: NoOp node  
    - Role: Ends flow for leads without valid emails; no action taken.

  - **Sticky Notes**  
    - Visual guides for email sending section and setup instructions including links and tutorial video.

---

### 3. Summary Table

| Node Name                | Node Type                       | Functional Role                         | Input Node(s)           | Output Node(s)               | Sticky Note                                                                                                                         |
|--------------------------|--------------------------------|---------------------------------------|------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| On form submission       | formTrigger                    | Capture user input from lead form     | -                      | HTTP Request                | # Business Data                                                                                                                    |
| HTTP Request             | httpRequest                   | Scrape lead data from Apify actor     | On form submission     | Filter                      |                                                                                                                                    |
| Filter                   | filter                       | Filter leads that have websites        | HTTP Request           | Append row in sheet, NoOp    |                                                                                                                                    |
| Append row in sheet      | googleSheets                 | Store filtered leads in Google Sheet  | Filter                 | Loop Over Items             | # Getting the Email Address                                                                                                        |
| Loop Over Items          | splitInBatches               | Iterate over stored leads              | Append row in sheet    | Wait, (empty branch)        | # Getting the Email Address                                                                                                        |
| Wait                     | wait                         | Delay between email generations        | Loop Over Items        | Information Extractor1      | # Email Send                                                                                                                       |
| Information Extractor    | langchain.informationExtractor (Google Gemini) | Extract email address from website    | Filter                 | If                         | # Getting the Email Address                                                                                                        |
| If                       | if                           | Check if extracted email is valid      | Information Extractor  | Append row in sheet, NoOp   |                                                                                                                                    |
| Information Extractor1   | langchain.informationExtractor (OpenAI GPT-4) | Generate cold email subject and body  | Wait                   | Edit Fields1                | # Email Send                                                                                                                       |
| Edit Fields1             | set                          | Add send time and email address info   | Information Extractor1  | Send a message              | # Email Send                                                                                                                       |
| Send a message           | gmail                        | Send cold email via Gmail              | Edit Fields1           | Append or update row in sheet | # Email Send                                                                                                                       |
| Append or update row in sheet | googleSheets             | Update lead status and timestamp       | Send a message         | Loop Over Items             | # Email Send                                                                                                                       |
| No Operation, do nothing | noOp                         | End flow for leads without valid email | Filter, If             | -                           |                                                                                                                                    |
| Google Gemini Chat Model | langchain.lmChatGoogleGemini | AI model for email extraction          | Information Extractor  | Information Extractor       |                                                                                                                                    |
| OpenAI Chat Model        | langchain.lmChatOpenAi       | AI model for cold email generation     | Information Extractor1 | Information Extractor1      |                                                                                                                                    |
| Sticky Note              | stickyNote                   | Visual notes for sections and guides   | -                      | -                           | Multiple sticky notes with setup guide and tutorial video links: https://youtu.be/3UwutV1x3mA                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: `formTrigger`  
   - Configure form titled "Lead Machine" with fields:  
     - Business Type (text, required)  
     - Location (text, required)  
     - Lead Number (number, required)  
     - Email Style (dropdown: Friendly, Professional, Simple)  
   - Set button label to "GO 🚀" and disable attribution.

2. **Add HTTP Request Node**  
   - Connect from Form Trigger  
   - Method: POST  
   - URL: Set to your Apify actor endpoint URL (replace placeholder `Apify_Actor_Endpoint_URL`)  
   - Body Type: JSON  
   - Body Content: Pass form data parameters dynamically:  
     - `locationQuery`: `{{ $json.Location }}`  
     - `maxCrawledPlacesPerSearch`: `{{ $json['Lead Number'] }}`  
     - `searchStringsArray`: `[ "{{ $json['Business Type'] }}" ]`  
     - Other scraping options set as per workflow.

3. **Add Filter Node**  
   - Connect from HTTP Request  
   - Condition: Check if `website` field exists and is non-empty (`exists`, strict, case-sensitive).  
   - True branch leads to Append row in sheet; False branch leads to No Operation.

4. **Add Google Sheets Append Row Node**  
   - Connect from Filter (true branch)  
   - Document ID: Your target Google Sheet ID  
   - Sheet Name: Sheet1 or your chosen sheet  
   - Map columns for company details (Company Name, Website, Phone Number, Address, Category) and empty Email Address field.

5. **Add SplitInBatches Node ("Loop Over Items")**  
   - Connect from Append row in sheet  
   - Default batch size or configure as needed.

6. **Add Wait Node**  
   - Connect from Loop Over Items  
   - Set delay to 1 second.

7. **Add Information Extractor Node (Google Gemini)**  
   - Connect from Wait  
   - Configure LangChain Information Extractor to extract "Email Address" from website text:  
     - Input: `Website: {{ $json.website }}`  
     - Attribute: "Email Address" required, description to find best formatted email address from website.

8. **Add If Node**  
   - Connect from Information Extractor (Google Gemini)  
   - Condition: Check if extracted "Email Address" contains "@"

9. **On If True branch:**  
   - Connect to Loop Over Items to continue processing  
   - Connect also to Information Extractor1 (OpenAI GPT-4 node)

10. **Add Information Extractor1 Node (OpenAI GPT-4)**  
    - Connect from Wait node (or from If true branch, depending on design)  
    - Configure prompt with company data and email style to generate "Mail Subject" and "Mail Body".  
    - Use GPT-4 mini or equivalent model.

11. **Add Set Node (Edit Fields1)**  
    - Connect from Information Extractor1  
    - Add fields:  
      - Send Time: current timestamp formatted as MM-dd-yyyy (h:mm a)  
      - Email Address: from Wait node data

12. **Add Gmail Send Node**  
    - Connect from Edit Fields1  
    - Configure to send email:  
      - To: `{{ $json['Email Address'] }}`  
      - Subject: `{{ $('Information Extractor1').item.json.output['Mail Subject'] }}`  
      - Message: `{{ $('Information Extractor1').item.json.output['Mail Body'] }}`  
      - Email type: text  
    - Set max tries to 2; continue on error.

13. **Add Google Sheets Append or Update Row Node**  
    - Connect from Gmail Send Node  
    - Configure to update lead row matching on "Email Address" with:  
      - Cold Mail Status: "✅"  
      - SEND Time: from Edit Fields1 node

14. **Add No Operation Node**  
    - Connect from Filter false branch and If false branch to end flow for leads without website or valid email.

15. **Setup Credentials:**  
    - Google Sheets OAuth2 with access to target sheet  
    - Gmail OAuth2 with send mail permission  
    - OpenAI API key for GPT-4 usage  
    - Google Gemini API key for email extraction

16. **Add Sticky Notes for Documentation:**  
    - Add visual sticky notes labeling blocks: Business Data, Getting the Email Address, Email Send, and Setup Guide with embedded tutorial video link: https://youtu.be/3UwutV1x3mA

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                           | Context or Link                                             |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| Video tutorial on creating this automated lead finder and cold email system with AI agents.                                                                                                                                          | https://youtu.be/3UwutV1x3mA                                |
| Setup guide notes embedded in workflow explain how to configure Apify endpoint, Google Gemini, Google Sheets, OpenAI, and Gmail integration.                                                                                         | Workflow sticky notes                                        |
| Workflow respects data privacy by selectively scraping publicly available business data and sending cold emails only after email address extraction and validity checks.                                                             | Workflow design principle                                   |
| Google Sheets columns required: Company Name, Category, Website, Phone Number, Email Address, Address, Cold Mail Status, SEND Time.                                                                                                   | Google Sheets configuration                                 |
| Use of AI models: Google Gemini for email extraction from website text; OpenAI GPT-4 for email content generation.                                                                                                                    | Node details                                                 |
| Retry and error handling: Gmail sending node retries twice and continues on failure; no operation nodes terminate flows cleanly for invalid data.                                                                                    | Node configurations                                         |

---

This document enables clear understanding, modification, and manual recreation of the Automated Lead Generation & Cold Emailing workflow using n8n, supporting both human operators and integration automation tools.