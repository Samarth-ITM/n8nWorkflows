Automated Local Lead Finder & Cold Email Sender with Apify, AI, and Gmail

https://n8nworkflows.xyz/workflows/automated-local-lead-finder---cold-email-sender-with-apify--ai--and-gmail-8986


# Automated Local Lead Finder & Cold Email Sender with Apify, AI, and Gmail

### 1. Workflow Overview

This workflow automates the process of finding local business leads and sending personalized cold emails using a combination of Apify web scraping, AI-powered email generation, and Gmail integration. It is designed for digital marketing agencies or sales teams aiming to scale outreach campaigns efficiently.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception & Lead Search:** Collect user input via a form (business type, location, lead count, email style) and query an Apify actor to scrape business leads.
- **1.2 Website Filtering & Email Extraction:** Filter leads to ensure they have websites, then extract the best email address from each website using AI.
- **1.3 Validation & Data Persistence:** Validate extracted emails and store lead details in a Google Sheet.
- **1.4 Outreach & Logging:** Loop through leads, generate personalized cold emails using AI, send emails via Gmail with pacing/waiting, and update the sheet with send status and timestamp.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Lead Search

**Overview:**  
This block begins the workflow with a form to collect user inputs concerning the lead search criteria, then sends these inputs to an Apify actor to retrieve lead data.

**Nodes Involved:**  
- On form submission  
- HTTP Request  
- Sticky Note (STEP 1 · Intake & Search)

**Node Details:**

- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point; captures user inputs: Business Type, Location, Lead Number, Email Style.  
  - Config: Form title "Lead Machine", required fields for business type, location, and lead number; dropdown for email style.  
  - Outputs form data as JSON for downstream use.  
  - Errors: Missing required fields block trigger; malformed inputs may affect HTTP request.  
  - Connections: Outputs to HTTP Request.

- **HTTP Request**  
  - Type: HTTP Request  
  - Role: Calls the Apify actor endpoint to scrape leads based on form inputs.  
  - Config: POST JSON body dynamically constructed using form inputs (e.g., location, business type, lead number).  
  - Key Expressions: Uses expressions like `{{ $json.Location }}`, `{{ $json['Lead Number'] }}`, and `{{ $json['Business Type'] }}` to populate search parameters.  
  - Output: Array of places with metadata (title, category, phone, website, address).  
  - Failure modes: HTTP errors, API rate limits, malformed response.  
  - Connections: Outputs to Filter node.

- **Sticky Note (STEP 1 · Intake & Search)**  
  - Describes the purpose of this block clearly for maintainers.

---

#### 2.2 Website Filtering & Email Extraction

**Overview:**  
Filters the scraped leads to only those with websites, then uses an AI-powered information extractor to find the best email address from each business website.

**Nodes Involved:**  
- Filter  
- Information Extractor  
- Google Gemini Chat Model (linked as AI model for extraction)  
- Sticky Note (STEP 2 · Website & Email Extraction)

**Node Details:**

- **Filter**  
  - Type: Filter  
  - Role: Ensures only leads containing a non-empty `website` field pass.  
  - Config: Condition checks if `website` field exists and is not empty.  
  - Failure: Empty or malformed website data causes exclusion.  
  - Connections: Passes filtered leads to Information Extractor.

- **Information Extractor**  
  - Type: LangChain Information Extractor  
  - Role: Extracts a single, best email address from the lead's website URL string.  
  - Config: Input text set as `Website: {{ $json.website }}`, attribute required is "Email Address".  
  - AI Model: Uses Google Gemini Chat Model as the underlying LLM.  
  - Failure: Extraction may fail if website lacks visible email or if AI misinterprets content.  
  - Connections: Output passes to If node.

- **Google Gemini Chat Model**  
  - Type: LangChain Google Gemini Chat Model  
  - Role: AI engine powering the Information Extractor node.  
  - Credential: Uses Google Palm API credentials.  
  - Failure: API call failure, quota exceeded, or latency issues.

- **Sticky Note (STEP 2 · Website & Email Extraction)**  
  - Notes importance of filtering and real inbox emails only.

---

#### 2.3 Validation & Data Persistence

**Overview:**  
Validates the extracted email to ensure it contains '@', then appends the lead data with email to a Google Sheet. This step avoids storing incomplete data.

**Nodes Involved:**  
- If  
- Append row in sheet (Google Sheets)  
- Sticky Note (STEP 3 · Validate & Persist)

**Node Details:**

- **If**  
  - Type: If condition  
  - Role: Checks if extracted email contains '@' to verify validity.  
  - Config: Condition uses string contains '@' on `output['Email Address']`.  
  - Failure: Invalid or missing emails are filtered out, not stored.  
  - Connections: Valid emails proceed to Append row in sheet.

- **Append row in sheet**  
  - Type: Google Sheets - Append  
  - Role: Persists lead data including company name, category, website, phone number, address, and extracted email.  
  - Config: Maps fields from filtered lead JSON and extracted email to corresponding Google Sheet columns.  
  - Credential: Google Sheets OAuth2 account.  
  - Failure: Sheet access errors, quota limits, or schema mismatches.  
  - Connections: Output connects to Loop Over Items for batch processing.

- **Sticky Note (STEP 3 · Validate & Persist)**  
  - Suggests setting matching columns in Sheets to avoid duplicates.

---

#### 2.4 Outreach & Logging

**Overview:**  
Iterates over stored leads, waits between sends to pace outreach, uses AI to generate personalized cold email subject and body according to email style, sends emails via Gmail, then updates the Sheet with send status and timestamp.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- Wait  
- Information Extractor1 (second instance)  
- OpenAI Chat Model  
- Edit Fields1 (Set node)  
- Send a message (Gmail)  
- Append or update row in sheet (Google Sheets)  
- Sticky Note (STEP 4 · Outreach & Logging)

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Iterates over lead rows to process each lead individually.  
  - Config: Default batch size (usually 1) to handle pacing.  
  - Failure: Batch processing errors, empty input data.  
  - Connections: Outputs to Wait node.

- **Wait**  
  - Type: Wait  
  - Role: Pauses workflow for 1 second between email sends to avoid rate limits or spam flags.  
  - Config: 1-second delay configured.  
  - Failure: Timeout issues if external trigger expected.  
  - Connections: Passes data to Information Extractor1.

- **Information Extractor1**  
  - Type: LangChain Information Extractor  
  - Role: Generates cold email subject and body using AI based on company data and chosen email style.  
  - Config: Inputs include company name, business type, and email style; instructions specify professional tone, greeting, signature.  
  - AI Model: Uses OpenAI GPT-4.1-mini as language model.  
  - Failure: AI generation errors, incomplete output.  
  - Connections: Output passes to Edit Fields1.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provides AI generation capabilities for Information Extractor1.  
  - Credential: OpenAI API key configured.  
  - Failure: API quota, latency, or network issues.

- **Edit Fields1**  
  - Type: Set  
  - Role: Adds metadata fields "Send Time" (current timestamp) and "Email Address" for logging.  
  - Config: Uses expression to format current date/time; pulls email from Wait node output.  
  - Failure: Expression evaluation errors.  
  - Connections: Output passes to Send a message.

- **Send a message**  
  - Type: Gmail  
  - Role: Sends the generated cold email to the extracted address.  
  - Config: Sends plain text email; subject and body taken from Information Extractor1 outputs.  
  - Credential: Gmail OAuth2 account configured.  
  - Failover: Retries up to 2 times; continues workflow on failure.  
  - Connections: Output passes to Append or update row in sheet.

- **Append or update row in sheet**  
  - Type: Google Sheets - AppendOrUpdate  
  - Role: Updates the lead row with "Cold Mail Status" marked as sent (✅) and logs the send time.  
  - Config: Uses email address as matching column to update existing row.  
  - Credential: Google Sheets OAuth2.  
  - Failure: Sheet access issues, concurrency conflicts.

- **Sticky Note (STEP 4 · Outreach & Logging)**  
  - Explains pacing, AI content generation, sending, and logging steps.

---

### 3. Summary Table

| Node Name              | Node Type                         | Functional Role                          | Input Node(s)          | Output Node(s)                   | Sticky Note                                                                                   |
|------------------------|----------------------------------|----------------------------------------|-----------------------|---------------------------------|----------------------------------------------------------------------------------------------|
| On form submission     | Form Trigger                     | Entry point, collects lead search info | -                     | HTTP Request                    | ## STEP 1 · Intake & Search: collects Business Type, Location, Lead Number, Email Style.     |
| HTTP Request           | HTTP Request                    | Calls Apify actor to scrape leads       | On form submission     | Filter                         | ## STEP 1 · Intake & Search: searches places using form inputs.                              |
| Filter                 | Filter                         | Filters leads with existing websites    | HTTP Request           | Information Extractor           | ## STEP 2 · Website & Email Extraction: ensures website exists before email extraction.       |
| Information Extractor  | LangChain Information Extractor | Extracts best email from website        | Filter                 | If                            | ## STEP 2 · Website & Email Extraction: find best real inbox email using AI.                  |
| Google Gemini Chat Model| LangChain AI Model             | AI engine for email extraction           | Information Extractor  | -                             |                                                                                              |
| If                     | If Condition                   | Validates email contains '@'             | Information Extractor  | Append row in sheet             | ## STEP 3 · Validate & Persist: filters invalid emails before saving to Google Sheets.       |
| Append row in sheet    | Google Sheets (Append)          | Stores lead details with email           | If                     | Loop Over Items                | ## STEP 3 · Validate & Persist: appends lead data to Google Sheets.                          |
| Loop Over Items        | SplitInBatches                 | Processes each lead individually         | Append row in sheet    | Wait                          | ## STEP 4 · Outreach & Logging: loops and paces email sending.                               |
| Wait                   | Wait                          | Adds delay between sends (1 second)      | Loop Over Items        | Information Extractor1         | ## STEP 4 · Outreach & Logging: pacing for rate limiting and spam avoidance.                 |
| Information Extractor1 | LangChain Information Extractor | Generates personalized cold email subject/body | Wait                   | Edit Fields1                  | ## STEP 4 · Outreach & Logging: drafts cold emails based on company data and email style.    |
| OpenAI Chat Model      | LangChain AI Model             | AI engine for cold email generation        | Information Extractor1 | -                             |                                                                                              |
| Edit Fields1           | Set                           | Adds send timestamp and email address    | Information Extractor1 | Send a message                | ## STEP 4 · Outreach & Logging: logs send time metadata.                                     |
| Send a message         | Gmail                         | Sends the cold email                      | Edit Fields1           | Append or update row in sheet  | ## STEP 4 · Outreach & Logging: sends email, retries on failure, continues workflow.         |
| Append or update row in sheet | Google Sheets (AppendOrUpdate) | Updates sheet with send status and time  | Send a message         | Loop Over Items (continue)     | ## STEP 4 · Outreach & Logging: marks cold mail status and logs send time in Google Sheets.  |
| Sticky Note            | Sticky Note                   | Documentation for STEP 1                  | -                     | -                             | ## STEP 1 · Intake & Search: Form Trigger and Apify actor details.                            |
| Sticky Note1           | Sticky Note                   | Documentation for STEP 2                  | -                     | -                             | ## STEP 2 · Website & Email Extraction: Filter and Information Extractor notes.               |
| Sticky Note2           | Sticky Note                   | Documentation for STEP 3                  | -                     | -                             | ## STEP 3 · Validate & Persist: If node and Append row info.                                 |
| Sticky Note3           | Sticky Note                   | Documentation for STEP 4                  | -                     | -                             | ## STEP 4 · Outreach & Logging: Loop, Wait, AI generation, Gmail sending, logging explained.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**  
   - Name: "On form submission"  
   - Configure form fields:  
     - Business Type (text, required)  
     - Location (text, required)  
     - Lead Number (number, required)  
     - Email Style (dropdown: Friendly, Professional, Simple)  
   - Set button label: "GO 🚀"  
   - Save and activate webhook.

2. **Add HTTP Request Node**  
   - Name: "HTTP Request"  
   - Method: POST  
   - URL: Set to your Apify actor endpoint URL (replace `"Apify_Actor_Endpoint_URL"` placeholder).  
   - Body Content Type: JSON  
   - JSON Body: Use expressions to pass form inputs, e.g.:  
     ```json
     {
       "includeWebResults": false,
       "language": "en",
       "locationQuery": "{{$json.Location}}",
       "maxCrawledPlacesPerSearch": {{$json['Lead Number']}},
       "maxImages": 0,
       "maximumLeadsEnrichmentRecords": 0,
       "scrapeContacts": false,
       "scrapeDirectories": false,
       "scrapeImageAuthors": false,
       "scrapePlaceDetailPage": false,
       "scrapeReviewsPersonalData": true,
       "scrapeTableReservationProvider": false,
       "searchStringsArray": ["{{$json['Business Type']}}"],
       "skipClosedPlaces": false
     }
     ```
   - Connect "On form submission" main output to HTTP Request input.

3. **Add Filter Node**  
   - Name: "Filter"  
   - Condition: Check if `website` field exists and is not empty (`string exists` on `{{$json.website}}`).  
   - Connect HTTP Request output to Filter input.

4. **Add LangChain Information Extractor Node**  
   - Name: "Information Extractor"  
   - Text input: `"Website: {{$json.website}}"`  
   - Define attribute: "Email Address", required, description to find the best single email from the website in ideal format.  
   - Connect Filter's true output to this node.

5. **Add Google Gemini Chat Model Node**  
   - Name: "Google Gemini Chat Model"  
   - Credential: Provide Google Palm API credentials.  
   - Connect as AI model to "Information Extractor" node.

6. **Add If Node**  
   - Name: "If"  
   - Condition: Check if extracted email (`{{$json.output['Email Address']}}`) contains '@'.  
   - Connect Information Extractor output to If input.

7. **Add Google Sheets Append Node**  
   - Name: "Append row in sheet"  
   - Configure with your Google Sheets document ID and Sheet1 (gid=0).  
   - Map columns: Address, Website, Category, Company Name, Email Address, Phone Number from the filtered lead JSON and extracted email.  
   - Connect If's true output to this node.

8. **Add SplitInBatches Node**  
   - Name: "Loop Over Items"  
   - Connect Append row in sheet output to this node.  
   - Default batch size (1).

9. **Add Wait Node**  
   - Name: "Wait"  
   - Wait time: 1 second.  
   - Connect Loop Over Items output to Wait input.

10. **Add Second LangChain Information Extractor Node**  
    - Name: "Information Extractor1"  
    - Text input template:  
      ```
      You are a perfect cold mail generator for a Digital Marketing Agency named Upward Engine.

      Here's the Information about the Recipient:

      Company Name: {{$json['Company Name']}}
      Business Type: {{$json.Category}}

      Email Style / Email Tune: {{$json['Email Style']}}

      Instructions:

      1. Always start with greeting the Company like Hi Company Name,
      2. Always use We not I.
      3. Mail must be professional, clean, and to the point.
      4. End with a signature like:
         [Your Name]
         [Your Company/Agency Name]
      ```
    - Define attributes:  
      - Mail Subject (required)  
      - Mail Body (required)  
    - Connect Wait output to this node.

11. **Add OpenAI Chat Model Node**  
    - Name: "OpenAI Chat Model"  
    - Model: Select GPT-4.1-mini or equivalent.  
    - Credential: Provide OpenAI API key.  
    - Connect as AI model to "Information Extractor1".

12. **Add Set Node**  
    - Name: "Edit Fields1"  
    - Add fields:  
      - Send Time: current timestamp formatted as "MM-dd-yyyy (h:mm a)" using expression: `{{$now.toFormat("MM-dd-yyyy (h:mm a)")}}`  
      - Email Address: from Wait node output `{{$json['Email Address']}}`  
    - Connect Information Extractor1 output to Set node.

13. **Add Gmail Node**  
    - Name: "Send a message"  
    - To: Set to `{{$json['Email Address']}}`  
    - Subject: Use `{{$json.output['Mail Subject']}}` from Information Extractor1  
    - Message Body: Use `{{$json.output['Mail Body']}}`  
    - Email Type: Plain text (switch to HTML if needed)  
    - Credential: Setup Gmail OAuth2 credentials.  
    - Retries: 2 attempts, on failure continue workflow.  
    - Connect Edit Fields1 output to Gmail node.

14. **Add Google Sheets Append or Update Node**  
    - Name: "Append or update row in sheet"  
    - Document and Sheet: Same as previous Sheets node.  
    - Matching Column: "Email Address" to update existing rows.  
    - Columns to update: "SEND Time" and "Cold Mail Status" with status "✅"  
    - Connect Gmail output to this node.

15. **Connect Append or update row output back to Loop Over Items input**  
    - This allows continuous processing of all leads in batches.

16. **Add Sticky Notes**  
    - Add notes at relevant visual positions to explain each step clearly for maintenance.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                     |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------|
| The workflow uses Apify actor for lead scraping—ensure your Apify endpoint is active and accessible. | Apify platform documentation: https://apify.com/docs              |
| AI models used: Google Gemini (PaLM) and OpenAI GPT-4.1-mini; ensure API keys and quotas are valid. | Google PaLM API: https://developers.generativeai.google           |
| Use Gmail OAuth2 credentials with proper scopes for sending emails via n8n Gmail node.              | Gmail API OAuth2 setup guide: https://developers.google.com/gmail/api/auth/about-auth |
| Consider Google Sheets API quota limits and avoid race conditions by setting matching columns.     | Google Sheets API: https://developers.google.com/sheets/api        |
| To avoid spam, pacing with Wait node is critical; adjust wait time per provider limitations.        | Email deliverability best practices: https://sendgrid.com/blog/email-deliverability/    |
| Customize email style options in the form to match your brand voice and campaign tone.              |                                                                     |

---

This structured breakdown enables advanced users and AI agents to understand, reproduce, and maintain the Automated Local Lead Finder & Cold Email Sender workflow fully and confidently.