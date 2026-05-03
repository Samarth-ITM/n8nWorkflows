Automated Meeting Attendee Enrichment with Apollo.io and Google Sheets

https://n8nworkflows.xyz/workflows/automated-meeting-attendee-enrichment-with-apollo-io-and-google-sheets-5892


# Automated Meeting Attendee Enrichment with Apollo.io and Google Sheets

---

### 1. Workflow Overview

This workflow automates the enrichment of meeting attendees' information by integrating meeting scheduling platforms (Cal.com and Calendly) with Apollo.io data scraping and Google Sheets for record keeping. The core use case is to capture newly scheduled meeting attendees, extract their basic info, enrich it with detailed professional data from Apollo.io, and log the enriched or fallback information into a Google Sheet for later reference.

**Logical Blocks:**

- **1.1 Input Reception:** Trigger nodes capture new booking or invitee creation events from Cal.com and Calendly.
- **1.2 Data Extraction:** Extract and normalize attendee information from the webhook payloads.
- **1.3 Logging Initial Data:** Log the extracted raw attendee data into Google Sheets.
- **1.4 Apollo Query Generation & URL Creation:** Build a structured query based on attendee info and generate a corresponding Apollo.io search URL.
- **1.5 Apollo Data Scraping:** Use Apify's Apollo scraper API to retrieve enriched data.
- **1.6 Conditional Handling & Final Logging:** Verify if enriched data is available; if yes, log enriched details, otherwise log fallback info with status "Info Not Available".

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
This block listens for new meeting scheduling events from two popular platforms, Cal.com and Calendly, triggering the workflow when new bookings or invitees are created.

**Nodes Involved:**  
- Cal.com Trigger1  
- Calendly Trigger  
- Sticky Note1 (resources and setup guide)

**Node Details:**

- **Cal.com Trigger1**  
  - *Type:* Cal.com Trigger  
  - *Role:* Listens for "BOOKING_CREATED" events from Cal.com via webhook.  
  - *Configurations:* Uses Cal.com API credential; webhook ID is preset.  
  - *Input:* Incoming webhook from Cal.com.  
  - *Output:* Emits booking data JSON.  
  - *Failure Modes:* Possible webhook misconfiguration, credential auth errors, or event mismatch.  
  - *Notes:* Requires Cal.com API key setup.

- **Calendly Trigger**  
  - *Type:* Calendly Trigger  
  - *Role:* Listens for "invitee.created" events from Calendly via webhook.  
  - *Configurations:* Uses Calendly API credential; webhook ID is preset.  
  - *Input:* Incoming webhook from Calendly.  
  - *Output:* Emits invitee data JSON.  
  - *Failure Modes:* Webhook setup errors, expired credentials, event mismatches.

- **Sticky Note1**  
  - *Role:* Provides resources including API key links, setup instructions, and useful templates.  
  - *Content:*  
    - Links to Calendly, Cal.com, Apify API key pages  
    - Google Sheets template link  
    - Detailed setup guide and support contact  
    - Link to additional real-world workflows

#### 1.2 Data Extraction

**Overview:**  
Transforms raw webhook payloads from Cal.com and Calendly into a normalized format with key attendee details (Name, Email, Company, Notes, Created At).

**Nodes Involved:**  
- Extract data (Cal.com path)  
- Extract Data (Calendly path)

**Node Details:**

- **Extract data**  
  - *Type:* Set  
  - *Role:* Maps Cal.com webhook JSON fields to normalized attributes: Name, Email, Company, Notes, Created At (converted to DateTime).  
  - *Key Expressions:* Uses `$json.responses.<field>.value` extraction for each attribute.  
  - *Input:* JSON from Cal.com webhook trigger.  
  - *Output:* JSON with normalized fields.  
  - *Failure Modes:* Missing or malformed webhook payload fields.

- **Extract Data**  
  - *Type:* Set  
  - *Role:* Maps Calendly webhook JSON fields similarly to the normalized structure.  
  - *Key Expressions:* Extracts from `$json.payload` and nested `questions_and_answers` arrays.  
  - *Input:* JSON from Calendly webhook trigger.  
  - *Output:* JSON with normalized fields.  
  - *Failure Modes:* Unexpected webhook schema changes or missing fields.

#### 1.3 Logging Initial Data

**Overview:**  
Logs the normalized attendee data into the Google Sheet as an initial record for tracking.

**Nodes Involved:**  
- Log entry

**Node Details:**

- **Log entry**  
  - *Type:* Google Sheets  
  - *Role:* Append or update a row with the attendee's basic info (Name, Email, Company, Notes, Created At).  
  - *Configuration:*  
    - Document and sheet IDs point to the designated Google Sheet ("Meeting Prep").  
    - Matching column: "Created At" to avoid duplicates.  
    - Uses Google Sheets OAuth2 credentials.  
  - *Input:* Normalized attendee data from extraction nodes.  
  - *Output:* Appended/updated Google Sheet row.  
  - *Failure Modes:* Google API quota/exceeded, auth token expiry, incorrect sheet or column mapping.

#### 1.4 Apollo Query Generation & URL Creation

**Overview:**  
Generates a structured search query to find enriched attendee data on Apollo.io and constructs a properly encoded Apollo search URL.

**Nodes Involved:**  
- Generate Query  
- Create URL

**Node Details:**

- **Generate Query**  
  - *Type:* Set  
  - *Role:* Builds a JSON object with query parameters using attendee's Name and Company.  
  - *Key Expressions:* Injects `Name` and `Company` from "Log entry" node into JSON fields `keyword` and `business` respectively.  
  - *Output:* JSON with the query array.  
  - *Failure Modes:* Missing input fields causing empty or malformed query.

- **Create URL**  
  - *Type:* Code (JavaScript)  
  - *Role:* Converts the JSON query into an Apollo.io search URL with correctly encoded parameters.  
  - *Implementation Highlights:*  
    - Processes arrays like locations and business keywords.  
    - Encodes parameters for URL safety (%20 for spaces, %5B%5D for brackets).  
    - Builds query string in a fixed parameter order expected by Apollo.  
  - *Input:* JSON query from "Generate Query".  
  - *Output:* Full Apollo search URL.  
  - *Failure Modes:* Code exceptions if input JSON structure is unexpected or empty.

#### 1.5 Apollo Data Scraping

**Overview:**  
Calls Apify's Apollo Scraper API to retrieve enriched contact data using the generated Apollo URL.

**Nodes Involved:**  
- Scrape Apollo

**Node Details:**

- **Scrape Apollo**  
  - *Type:* HTTP Request  
  - *Role:* Sends a POST request to Apify's Apollo scraper endpoint with parameters including the generated search URL to fetch enriched data.  
  - *Configuration:*  
    - URL: Apify actor run-sync endpoint with a token placeholder `<YOURAPIKEY>`.  
    - JSON body includes count=25 results, flags to include emails, and the `searchUrl` from "Create URL".  
    - Uses POST method with JSON body.  
  - *Input:* Final Apollo URL from "Create URL".  
  - *Output:* JSON array of enriched contact data.  
  - *Failure Modes:*  
    - Invalid or expired Apify API key.  
    - Network timeouts.  
    - Empty results or API errors.  
    - Rate limiting.

#### 1.6 Conditional Handling & Final Logging

**Overview:**  
Checks if enriched data was successfully retrieved. If yes, logs the enriched details; if no, logs a fallback entry with "Info Not Available" status.

**Nodes Involved:**  
- If Data available?  
- Google Sheets1 (Enriched data logging)  
- Google Sheets2 (Fallback logging)

**Node Details:**

- **If Data available?**  
  - *Type:* If  
  - *Role:* Checks if the output from "Scrape Apollo" node is non-empty (stringified JSON is not empty).  
  - *Input:* Data from "Scrape Apollo".  
  - *Output:*  
    - True branch if data present → "Google Sheets1"  
    - False branch if empty → "Google Sheets2"  
  - *Failure Modes:* Incorrect condition logic leading to false negatives.

- **Google Sheets1**  
  - *Type:* Google Sheets  
  - *Role:* Append or update Google Sheet with enriched attendee data including phone, company industry, socials, job title, website, company size, status "Enriched", etc.  
  - *Configuration:* Similar to "Log entry" but with extended fields and using data directly from Apollo scraper output.  
  - *Failure Modes:* Google API quota, data mapping errors.

- **Google Sheets2**  
  - *Type:* Google Sheets  
  - *Role:* Append or update fallback entry with status "Info Not Available" and minimal data.  
  - *Failure Modes:* Same as above.

---

### 3. Summary Table

| Node Name         | Node Type           | Functional Role                           | Input Node(s)       | Output Node(s)      | Sticky Note                                                                                 |
|-------------------|---------------------|-----------------------------------------|---------------------|---------------------|---------------------------------------------------------------------------------------------|
| Cal.com Trigger1   | Cal.com Trigger     | Trigger on Cal.com booking creation     | None (webhook)      | Extract data        |                                                                                             |
| Calendly Trigger   | Calendly Trigger    | Trigger on Calendly invitee creation    | None (webhook)      | Extract Data        |                                                                                             |
| Extract data       | Set                 | Normalize Cal.com webhook attendee data | Cal.com Trigger1    | Log entry           |                                                                                             |
| Extract Data       | Set                 | Normalize Calendly webhook attendee data| Calendly Trigger    | Log entry           |                                                                                             |
| Log entry          | Google Sheets       | Log initial attendee data                | Extract data / Extract Data | Generate Query    |                                                                                             |
| Generate Query     | Set                 | Build Apollo search query JSON           | Log entry           | Create URL          |                                                                                             |
| Create URL        | Code (JavaScript)    | Generate Apollo.io search URL             | Generate Query      | Scrape Apollo       |                                                                                             |
| Scrape Apollo      | HTTP Request        | Query Apify Apollo scraper API            | Create URL          | If Data available?   |                                                                                             |
| If Data available? | If                  | Check presence of enriched data           | Scrape Apollo       | Google Sheets1 / Google Sheets2 |                                                                                             |
| Google Sheets1     | Google Sheets       | Log enriched attendee data                | If Data available? (true) | None            |                                                                                             |
| Google Sheets2     | Google Sheets       | Log fallback attendee data (no enrichment)| If Data available? (false) | None            |                                                                                             |
| Sticky Note        | Sticky Note         | Title: "Enrich Meeting Attendees"         | None                | None                |                                                                                             |
| Sticky Note1       | Sticky Note         | Resources and setup guide                  | None                | None                | Contains resources and setup links for Calendly, Cal.com, Apify, Google Sheets, and more.  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**

   - Add **Cal.com Trigger** node:  
     - Set event to `BOOKING_CREATED`.  
     - Configure Cal.com API credentials with your API key.  
     - Save webhook URL and configure it in Cal.com.

   - Add **Calendly Trigger** node:  
     - Set event to `invitee.created`.  
     - Configure Calendly API credentials with your API key.  
     - Save webhook URL and configure it in Calendly.

2. **Create Data Extraction Nodes:**

   - Add **Set** node named "Extract data" connected to Cal.com Trigger:  
     - Map fields from Cal.com webhook JSON:  
       - Name: `{{$json.responses.name.value}}`  
       - Email: `{{$json.responses.email.value}}`  
       - Company: `{{$json.responses.title.value}}`  
       - Notes: `{{$json.responses.notes.value}}`  
       - Created at: `{{$json.createdAt.toDateTime()}}`

   - Add **Set** node named "Extract Data" connected to Calendly Trigger:  
     - Map fields from Calendly webhook JSON:  
       - Name: `{{$json.payload.name}}`  
       - Email: `{{$json.payload.email}}`  
       - Company: `{{$json.payload.questions_and_answers[0].answer}}`  
       - Notes: `{{$json.payload.questions_and_answers[1].answer}}`  
       - Created at: `{{$json.created_at.toDateTime()}}`

3. **Log Initial Data:**

   - Add **Google Sheets** node named "Log entry" connected to both extraction nodes:  
     - Operation: Append or update.  
     - Document ID: Your Google Sheet document ID.  
     - Sheet name: The sheet (e.g., "Sheet1" or gid=0).  
     - Matching column: "Created At" to avoid duplicates.  
     - Map columns: Email, Name, Notes, Company, Created At as per extracted data.  
     - Configure OAuth2 credentials for Google Sheets.

4. **Generate Apollo Query:**

   - Add **Set** node named "Generate Query" connected to "Log entry":  
     - Use raw JSON mode with this template:  
       ```json
       {
         "query": [
           {
             "keyword": ["{{$json['Name ']}}"],
             "business": ["{{$json.Company}}"]
           }
         ]
       }
       ```

5. **Create Apollo Search URL:**

   - Add **Code** node named "Create URL" connected to "Generate Query":  
     - Implement JavaScript code to build the Apollo search URL encoding:  
       - Base URL: `https://app.apollo.io/#/people`  
       - Include parameters: page=1, personLocations[], qOrganizationKeywordTags[], includedOrganizationKeywordFields[], sortByField, sortAscending, qKeywords, etc.  
       - Use encodeURIComponent for safety.  
       - Return `{ finalURL: <constructed URL> }`

6. **Call Apollo Scraper API:**

   - Add **HTTP Request** node named "Scrape Apollo" connected to "Create URL":  
     - Method: POST  
     - URL: `https://api.apify.com/v2/acts/supreme_coder~apollo-scraper/run-sync-get-dataset-items?token=<YOURAPIKEY>` (replace `<YOURAPIKEY>` with your Apify token)  
     - Body (JSON):  
       ```json
       {
         "count": 25,
         "excludeGuessedEmails": false,
         "excludeNoEmails": false,
         "getEmails": true,
         "searchUrl": "{{$json.finalURL}}"
       }
       ```  
     - Ensure "Send Body" is enabled and content type JSON.

7. **Check Data Availability:**

   - Add **If** node named "If Data available?" connected to "Scrape Apollo":  
     - Condition: Check if `{{ $node["Scrape Apollo"].all().toJsonString() }}` is not empty.

8. **Log Enriched Data or Fallback:**

   - Add **Google Sheets** node named "Google Sheets1" (for enriched data) connected to If node's **true** branch:  
     - Configure as "Append or Update".  
     - Map enriched fields from Apollo scraper output: Phone, Company, Industry, Job Title, Socials (LinkedIn, Twitter, etc.), Company Size, Status = "Enriched", etc.  
     - Use the same Google Sheets document and sheet as before.  
     - Use OAuth2 credentials.

   - Add **Google Sheets** node named "Google Sheets2" (for fallback) connected to If node's **false** branch:  
     - Append or update with minimal info and Status = "Info Not Available".  
     - Map Created At and Status fields.

9. **Add Sticky Notes (Optional):**

   - Add Sticky Note nodes for workflow title: "Enrich Meeting Attendees" and resource/setup instructions with links for API keys, templates, and guides.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                            | Context or Link                                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------|
| Resources including API key links for Calendly, Cal.com, Apify; Google Sheets template; detailed setup guide PDF; and support email contact. Additional links to real-world workflows document.                                                                                                                                                                                                                 | Sticky Note1 content; useful for onboarding and configuration.                                                          |
| Google Sheets template used as the main data store for logging meeting attendee information, enriched or fallback.                                                                                                                                                                                                                                                                                      | Google Sheets document ID referenced in nodes "Log entry", "Google Sheets1", "Google Sheets2".                         |
| Apify Apollo Scraper actor is an external service requiring a valid API key; replace `<YOURAPIKEY>` with your actual token.                                                                                                                                                                                                                                                                            | Critical for scraping enriched data from Apollo.io.                                                                     |
| The workflow is compatible with n8n versions supporting latest Google Sheets OAuth2 node (v4.6), HTTP Request, Set, If, and Cal/Calendly trigger nodes.                                                                                                                                                                                                                                                 | Version-specific note to avoid compatibility issues.                                                                    |
| The "Create URL" code node carefully encodes URL parameters to ensure Apollo.io search URL correctness, including bracket encoding for array parameters and space encoding as %20 instead of '+'.                                                                                                                                                                                                        | Important to maintain to avoid malformed URLs or failed API queries.                                                    |

---

**Disclaimer:** The provided content originates solely from an automated workflow created using n8n, an integration and automation tool. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.

---