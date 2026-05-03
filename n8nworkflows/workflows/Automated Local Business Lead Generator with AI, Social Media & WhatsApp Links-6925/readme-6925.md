Automated Local Business Lead Generator with AI, Social Media & WhatsApp Links

https://n8nworkflows.xyz/workflows/automated-local-business-lead-generator-with-ai--social-media---whatsapp-links-6925


# Automated Local Business Lead Generator with AI, Social Media & WhatsApp Links

### 1. Workflow Overview

This workflow automates the generation and management of local business leads by integrating data extraction from Google Maps, AI-powered business analysis, social media profile detection, and communication link generation. It targets digital marketing professionals and agencies aiming to efficiently gather comprehensive business data, analyze online presence, and engage business owners with personalized messages.

**Logical Blocks:**

- **1.1 Scheduled Input & Data Retrieval:** Triggered by a scheduled event, retrieves keywords and location data from Google Sheets to initiate searches.
- **1.2 Google Maps Data Extraction:** Uses SerpAPI to fetch local business data from Google Maps based on keywords and locations.
- **1.3 Data Processing & Storage:** Splits the retrieved business list, stores each item in Google Sheets, and filters entries requiring further processing.
- **1.4 AI-Powered Business Review Summarization:** Limits processing to batches, generates AI summaries of business reviews, and updates the sheet.
- **1.5 Website Content Analysis & Social Media Profile Extraction:** For businesses with websites, fetches the site HTML, extracts Instagram and TikTok links, summarizes web content with AI, and updates Google Sheets.
- **1.6 Notifications & Outreach Message Delivery:** Sends Telegram messages with latest lead data for follow-up.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Input & Data Retrieval

**Overview:**  
This block initiates the workflow on a schedule, fetching search parameters (keywords, location, country codes) from a Google Sheet that contains the input data for lead generation.

**Nodes Involved:**  
- Schedule Trigger  
- Get row(s) in sheet  

**Node Details:**

- **Schedule Trigger**  
  - Type: Trigger (Schedule)  
  - Configuration: Runs periodically based on user-defined interval (default is every minute/hour/day depending on setup).  
  - Input/Output: No input; output triggers downstream nodes.  
  - Edge Cases: Misconfiguration can cause excessive API calls or no triggers.  
  - Version: Standard  

- **Get row(s) in sheet**  
  - Type: Google Sheets  
  - Configuration: Reads rows from the first sheet (sheetName: "data") of the configured Google Sheet document. Retrieves keyword, city/location, and country code for queries.  
  - Input: Trigger from Schedule Trigger  
  - Output: Provides JSON data with search parameters for the HTTP Request node.  
  - Edge Cases: Sheet permission errors, empty rows, or missing columns can cause failures.  
  - Credential: Google Sheets OAuth2  

---

#### 1.2 Google Maps Data Extraction

**Overview:**  
Queries SerpAPI’s Google Maps engine to retrieve local business data such as name, address, rating, reviews, phone, website, and business types based on input keywords and location.

**Nodes Involved:**  
- HTTP Request  
- Split Out  

**Node Details:**

- **HTTP Request**  
  - Type: HTTP Request  
  - Configuration: Calls SerpAPI’s search endpoint with parameters: api_key, engine = google_maps, type = search, google_domain, query composed of keyword + city/location, and country code (gl).  
  - Input: From "Get row(s) in sheet"  
  - Output: JSON response containing local_results array with business data.  
  - Edge Cases: API key invalid or quota exceeded, network timeouts, malformed queries.  
  - Version: 4.2  

- **Split Out**  
  - Type: Split Out  
  - Configuration: Splits the JSON array field "local_results" into individual items to process each business separately.  
  - Input: HTTP Request response data  
  - Output: Multiple items, one per business.  
  - Edge Cases: Empty results array leads to no further processing.  
  - Execute Once: true  

---

#### 1.3 Data Processing & Storage

**Overview:**  
Stores individual business entries into a Google Sheet, then fetches stored rows for further operations. Filters out already summarized entries.

**Nodes Involved:**  
- Append or update row in sheet  
- Get row(s) in sheet1  
- If  

**Node Details:**

- **Append or update row in sheet**  
  - Type: Google Sheets  
  - Configuration: Writes or updates business data rows in a "results" sheet using "data_id" as the matching column. Creates WhatsApp link from phone number by stripping non-digit characters. Stores fields like title, types, rating, address, reviews, website, reviews_link, and phone.  
  - Input: From "Split Out"  
  - Output: Updated sheet row confirmation  
  - Edge Cases: Sheet permission issues, duplicate IDs, or invalid phone numbers.  
  - Credential: Google Sheets OAuth2  

- **Get row(s) in sheet1**  
  - Type: Google Sheets  
  - Configuration: Reads the "results" sheet to get all stored business entries for filtering and further analysis.  
  - Input: From "Append or update row in sheet"  
  - Output: List of all stored business entries  
  - Execute Once: true  

- **If**  
  - Type: If (conditional)  
  - Configuration: Checks if the "review_summary" field is empty for each business row to identify entries needing AI review summarization.  
  - Input: From "Get row(s) in sheet1"  
  - Output: Two branches: yes (empty summary) and no (already summarized)  
  - Edge Cases: Field case sensitivity or missing fields may cause logic errors.  

---

#### 1.4 AI-Powered Business Review Summarization

**Overview:**  
Limits the number of businesses processed in a batch to 5 to optimize resource usage, then sends business info to an AI model (OpenRouter) for generating a professional review summary. Updates the sheet with the summary.

**Nodes Involved:**  
- Limit  
- Basic LLM Chain  
- Append or update row in sheet3  

**Node Details:**

- **Limit**  
  - Type: Limit  
  - Configuration: Restricts processing to the first 5 business items per run.  
  - Input: From "If" node’s true branch  
  - Output: Limited subset of businesses for AI processing  

- **Basic LLM Chain**  
  - Type: Langchain LLM Chain  
  - Configuration: Uses OpenRouter Chat Model with a prompt that includes business information (name, address, rating, reviews count, types, phone, website, reviews link). Generates a professional message to the business owner including positioning, strengths, improvement areas, and a call to action in the local language (based on country code).  
  - Input: From "Limit"  
  - Output: Generated text summary  
  - Edge Cases: API limits, prompt errors, language model failures.  
  - Sub-workflow: Uses OpenRouter Chat Model node as language model backend.  

- **Append or update row in sheet3**  
  - Type: Google Sheets  
  - Configuration: Updates the "results" sheet with the generated "review_summary" text, matching on "data_id".  
  - Input: From "Basic LLM Chain"  
  - Output: Confirmation of sheet update  

---

#### 1.5 Website Content Analysis & Social Media Profile Extraction

**Overview:**  
For businesses with a website URL, fetches the website HTML, extracts Instagram and TikTok links via regular expressions, summarizes key website content using an AI prompt, and stores results in Google Sheets.

**Nodes Involved:**  
- If has a web  
- HTTP Request1  
- Code1  
- Code  
- Basic LLM Chain1  
- Append or update row in sheet1  
- Append or update row in sheet2  

**Node Details:**

- **If has a web**  
  - Type: If (conditional)  
  - Configuration: Checks if the "website" field is present and non-empty in the business data.  
  - Input: From "Append or update row in sheet3"  
  - Output: Yes branch leads to web analysis; No branch triggers Telegram notification.  

- **HTTP Request1**  
  - Type: HTTP Request  
  - Configuration: Fetches raw HTML content from the business’s website URL.  
  - Input: From "If has a web" yes branch  
  - Output: Raw HTML data  

- **Code1**  
  - Type: Code (JavaScript)  
  - Configuration: Parses HTML to extract page title, meta description, keywords, headings (h1-h4), paragraphs (filtered by length), and CTA buttons (links/buttons with keywords like order, buy, contact). Produces a simplified summary string.  
  - Input: From "HTTP Request1"  
  - Output: JSON with simplified_summary field  

- **Code**  
  - Type: Code (JavaScript)  
  - Configuration: Uses regex to extract first Instagram and TikTok profile URLs from the HTML content. Returns extracted URLs or null if not found.  
  - Input: From "HTTP Request1"  

- **Basic LLM Chain1**  
  - Type: Langchain LLM Chain  
  - Configuration: Sends website content summary to OpenRouter Chat Model to generate a professional message to the business owner with analysis and improvement suggestions, localized by country code.  
  - Input: From "Code1"  
  - Output: Text message for outreach  

- **Append or update row in sheet1**  
  - Type: Google Sheets  
  - Configuration: Updates Instagram and TikTok profile URLs for the business in the "results" sheet.  
  - Input: From "Code"  

- **Append or update row in sheet2**  
  - Type: Google Sheets  
  - Configuration: Updates the "web_summary" field in "results" sheet with the AI-generated website analysis message.  
  - Input: From "Basic LLM Chain1"  

---

#### 1.6 Notifications & Outreach Message Delivery

**Overview:**  
Sends Telegram messages with the latest processed business data including WhatsApp link for easy follow-up by marketing personnel.

**Nodes Involved:**  
- Send a text message  
- Send a text message1  

**Node Details:**

- **Send a text message**  
  - Type: Telegram  
  - Configuration: Sends a formatted message with business name, types, location, and WhatsApp link (phone number) to a configured Telegram chat ID. Intended for businesses without websites.  
  - Input: From "If has a web" no branch  

- **Send a text message1**  
  - Type: Telegram  
  - Configuration: Sends a similar message as above for businesses that have website analysis completed.  
  - Input: From "Append or update row in sheet2"  

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                 | Input Node(s)               | Output Node(s)                 | Sticky Note                                                                                                      |
|-----------------------------|----------------------------------|------------------------------------------------|-----------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger                 | Initiates workflow on schedule                  |                             | Get row(s) in sheet            | - Run workflows automatically based on a specified schedule. - Fetch data from a Google Sheet containing a list of keywords and search locations. - Send requests to SerpAPI to retrieve business data from Google Maps based on input from the Sheet. |
| Get row(s) in sheet        | Google Sheets                   | Fetches search keywords and locations           | Schedule Trigger            | HTTP Request                  |                                                                                                                 |
| HTTP Request               | HTTP Request                   | Queries SerpAPI Google Maps for business data   | Get row(s) in sheet        | Split Out                     |                                                                                                                 |
| Split Out                  | Split Out                      | Splits array of businesses into individual items| HTTP Request               | Append or update row in sheet | - Break the local results array from the SerpAPI JSON output into individual items for individual processing. - Save each business result to a Google Sheet (results tab), using the ID data as the matching column. - Retrieve data from the results sheet for further analysis or processing. - Filter unsummarized data for further processing. |
| Append or update row in sheet | Google Sheets                   | Stores individual business data in Google Sheets| Split Out                  | Get row(s) in sheet1          |                                                                                                                 |
| Get row(s) in sheet1       | Google Sheets                   | Retrieves stored business data                   | Append or update row in sheet | If                         |                                                                                                                 |
| If                         | If                             | Filters businesses missing review summaries     | Get row(s) in sheet1       | Limit, No Operation           |                                                                                                                 |
| Limit                      | Limit                          | Limits batch size for AI processing              | If (true branch)            | Basic LLM Chain               | - Limit the number of items submitted to LLM for efficiency (e.g., only 5 businesses per batch) - Generate a review summary using a business information-based prompt. - The AI model used by the LLM Chain node above. - Save the review_summary results to Google Sheets (results tab), matching based on ID data. - Check if the business has a website for further processing. |
| Basic LLM Chain            | Langchain LLM Chain            | Generates AI review summary message              | Limit                      | Append or update row in sheet3 |                                                                                                                 |
| Append or update row in sheet3 | Google Sheets                   | Saves AI review summary to sheet                  | Basic LLM Chain            | If has a web                 |                                                                                                                 |
| If has a web               | If                             | Checks if business has a website                  | Append or update row in sheet3 | HTTP Request1, Send a text message |                                                                                                                 |
| HTTP Request1              | HTTP Request                   | Fetches business website HTML                     | If has a web (true)        | Code1, Code                  | - Fetch HTML content from a business website. - Extract important elements from the HTML using Regex, then organize them into a ready to use summary. - Provide an AI model for generating summaries. |
| Code1                      | Code                           | Extracts structured summary elements from HTML  | HTTP Request1              | Basic LLM Chain1             |                                                                                                                 |
| Code                       | Code                           | Extracts Instagram and TikTok URLs from HTML     | HTTP Request1              | Append or update row in sheet1 | - Find Instagram and TikTok account links from business website pages. - Save Instagram and TikTok extraction results to Google Sheets. |
| Basic LLM Chain1           | Langchain LLM Chain            | Generates AI message based on website content    | Code1                      | Append or update row in sheet2 |                                                                                                                 |
| Append or update row in sheet1 | Google Sheets                   | Updates Instagram and TikTok profiles             | Code                       |                             |                                                                                                                 |
| Append or update row in sheet2 | Google Sheets                   | Saves AI website analysis message                  | Basic LLM Chain1           | Send a text message1         |                                                                                                                 |
| Send a text message        | Telegram                       | Sends Telegram notification for businesses without website | If has a web (false)        |                             |                                                                                                                 |
| Send a text message1       | Telegram                       | Sends Telegram notification for businesses with website analysis | Append or update row in sheet2 |                             |                                                                                                                 |
| No Operation, do nothing   | No Operation                   | Handles negative condition path                   | If (false branch)          |                             |                                                                                                                 |
| OpenRouter Chat Model      | Langchain Language Model       | Provides AI backend for Basic LLM Chain           |                            | Basic LLM Chain             |                                                                                                                 |
| OpenRouter Chat Model1     | Langchain Language Model       | Provides AI backend for Basic LLM Chain1          |                            | Basic LLM Chain1            |                                                                                                                 |
| Sticky Note                | Sticky Note                   | General workflow description and requirements     |                            |                             | # Automated Local Business Lead Generator with AI, Social Media & WhatsApp Links... See details in workflow notes. |
| Sticky Note1               | Sticky Note                   | Explains initial data retrieval block             |                            |                             | - Run workflows automatically based on a specified schedule. - Fetch data from a Google Sheet containing a list of keywords and search locations. - Send requests to SerpAPI to retrieve business data from Google Maps based on input from the Sheet. |
| Sticky Note2               | Sticky Note                   | Explains business data processing and storage     |                            |                             | - Break the local results array from the SerpAPI JSON output into individual items for individual processing. - Save each business result to a Google Sheet (results tab), using the ID data as the matching column. - Retrieve data from the results sheet for further analysis or processing. - Filter unsummarized data for further processing. |
| Sticky Note3               | Sticky Note                   | Explains AI review summarization block             |                            |                             | - Limit the number of items submitted to LLM for efficiency (e.g., only 5 businesses per batch) - Generate a review summary using a business information-based prompt. - The AI model used by the LLM Chain node above. - Save the review_summary results to Google Sheets (results tab), matching based on ID data. - Check if the business has a website for further processing. |
| Sticky Note4               | Sticky Note                   | Explains website content fetching and summarization |                            |                             | - Fetch HTML content from a business website. - Extract important elements from the HTML using Regex, then organize them into a ready to use summary. - Provide an AI model for generating summaries. |
| Sticky Note5               | Sticky Note                   | Explains data saving and Telegram notification     |                            |                             | - Save analysis results to Google Sheets. - Send automatic notifications to Telegram once data is processed and saved. |
| Sticky Note6               | Sticky Note                   | Explains social media profile extraction           |                            |                             | - Find Instagram and TikTok account links from business website pages. - Save Instagram and TikTok extraction results to Google Sheets. |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure interval as desired (e.g., every hour).  
   - Connect output to next node.

2. **Create Google Sheets Node ("Get row(s) in sheet")**  
   - Operation: Read rows  
   - Sheet: Select your Google Sheet and "data" tab (gid=0)  
   - Connect input from Schedule Trigger.  
   - Ensure Google Sheets credentials are configured.

3. **Create HTTP Request Node ("HTTP Request")**  
   - Method: GET  
   - URL: `https://serpapi.com/search`  
   - Query parameters:
     - `api_key`: Your SerpAPI key  
     - `engine`: "google_maps"  
     - `type`: "search"  
     - `google_domain`: "google.com"  
     - `q`: Expression combining `{{$json.keyword}} {{$json['city or specific location']}}`  
     - `gl`: `{{$json['country code']}}`  
   - Connect input from Google Sheets node.  

4. **Create Split Out Node**  
   - Field to split out: `local_results`  
   - Connect input from HTTP Request.

5. **Create Google Sheets Node ("Append or update row in sheet")**  
   - Operation: Append or update  
   - Sheet: Choose your "results" tab  
   - Matching Column: `data_id`  
   - Map columns: title, address, rating, reviews, reviews_link, types, phone, website, data_id, and a WhatsApp link computed as `"https://wa.me/" + phone digits only`  
   - Connect input from Split Out node.

6. **Create Google Sheets Node ("Get row(s) in sheet1")**  
   - Operation: Read rows  
   - Sheet: "results" tab  
   - Connect input from previous Google Sheets node.  
   - Set "Execute Once" to true.

7. **Create If Node**  
   - Condition: Check if `review_summary` field is empty  
   - Input from "Get row(s) in sheet1" node.  
   - True branch proceeds to Limit node; False branch to No Operation.

8. **Create Limit Node**  
   - Max items: 5  
   - Connect input from If's true branch.

9. **Create Langchain LLM Chain Node ("Basic LLM Chain")**  
   - Set prompt with business info fields (title, address, rating, reviews, types, phone, website, reviews_link)  
   - Add instructions to generate a professional message including summary, strengths, improvements, suggestion, and CTA in local language based on country code.  
   - Connect input from Limit node.  
   - Configure to use OpenRouter Chat Model as language model backend.

10. **Create Google Sheets Node ("Append or update row in sheet3")**  
    - Operation: Append or update  
    - Sheet: "results" tab  
    - Matching Column: `data_id`  
    - Update `review_summary` with AI output.  
    - Connect input from Basic LLM Chain.

11. **Create If Node ("If has a web")**  
    - Condition: Check if `website` field is not empty for each business.  
    - Input from "Append or update row in sheet3".  
    - True branch proceeds to HTTP Request1; False branch to Telegram notification node.

12. **Create HTTP Request Node ("HTTP Request1")**  
    - Method: GET  
    - URL: Use `website` field dynamically  
    - Connect input from "If has a web" true branch.

13. **Create Code Node ("Code1")**  
    - JavaScript code to extract title, meta description, keywords, headings (h1-h4), paragraphs (filtered), and CTA buttons from HTML.  
    - Connect input from HTTP Request1.

14. **Create Code Node ("Code")**  
    - JavaScript code to extract Instagram and TikTok URLs from HTML.  
    - Connect input from HTTP Request1.

15. **Create Langchain LLM Chain Node ("Basic LLM Chain1")**  
    - Set prompt with business name, website URL and content summary from Code1 output.  
    - Ask AI to produce a professional message with suggestions and CTA, localized by country code.  
    - Connect input from Code1.  
    - Use OpenRouter Chat Model as backend.

16. **Create Google Sheets Node ("Append or update row in sheet1")**  
    - Operation: Append or update  
    - Sheet: "results" tab  
    - Matching Column: `data_id`  
    - Update Instagram and TikTok profile URLs from Code output.  
    - Connect input from Code.

17. **Create Google Sheets Node ("Append or update row in sheet2")**  
    - Operation: Append or update  
    - Sheet: "results" tab  
    - Matching Column: `data_id`  
    - Update `web_summary` with AI output from Basic LLM Chain1.  
    - Connect input from Basic LLM Chain1.

18. **Create Telegram Node ("Send a text message")**  
    - Type: Telegram  
    - Chat ID: Your Telegram chat ID  
    - Text: Compose message with business name, types, location, and WhatsApp link.  
    - Connect input from "If has a web" false branch.

19. **Create Telegram Node ("Send a text message1")**  
    - Similar to above, for businesses with website analysis completed.  
    - Connect input from "Append or update row in sheet2".

20. **Create No Operation Node ("No Operation, do nothing")**  
    - Connect input from If node’s false branch (businesses with review_summary already present).  

21. **Create OpenRouter Chat Model nodes for both LLM Chain nodes.**  
    - Configure with your OpenRouter API key and model "google/gemini-2.0-flash-exp:free".  
    - Connect as language model backend to the respective LLM Chain nodes.

22. **Create Sticky Notes (Optional but recommended)**  
    - Add notes with descriptions, links to Google Sheets template, and instructions for each block.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                             | Context or Link                                                                                                                        |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| This workflow automates local business lead generation using SerpAPI, OpenRouter AI, Google Sheets, and Telegram for notifications. It includes social media detection and WhatsApp link creation for better outreach.                                                                                                      | See main Sticky Note in workflow for detailed description and project overview.                                                       |
| Use the provided Google Sheets template to ensure correct data structure: [Local Business Lead Generator](https://docs.google.com/spreadsheets/d/1s1N_cAFoKtCsolQh4v3QZpqr8KmVzi7agKHr5MdBEBs/edit?usp=sharing)                                                                                                          | Google Sheets Template                                                                                                                |
| Required credentials: SerpAPI API key, OpenRouter API key, Telegram Bot token, Google Sheets OAuth2. Configure each credential properly in n8n before running.                                                                                                                                                             | Credential setup                                                                                                                     |
| The workflow generates messages in the local language based on the country code provided in the input sheet, leveraging OpenRouter's language understanding capabilities.                                                                                                                                               | Language localization feature                                                                                                        |
| Telegram notifications help the marketing team to follow up leads quickly with essential business info and direct WhatsApp links, improving operational efficiency.                                                                                                                                                        | Telegram integration                                                                                                                 |
| The workflow is designed to be scalable and can be adapted to different niches, locations, and business types by modifying the input Google Sheet and prompts.                                                                                                                                                            | Customization notes                                                                                                                  |

---

**Disclaimer:**  
The provided text is exclusively derived from an automated n8n workflow, respecting all content policies and handling only legal and public data. No illegal or protected content is processed.