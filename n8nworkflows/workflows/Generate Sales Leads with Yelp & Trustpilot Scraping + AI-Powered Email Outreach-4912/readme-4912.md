Generate Sales Leads with Yelp & Trustpilot Scraping + AI-Powered Email Outreach

https://n8nworkflows.xyz/workflows/generate-sales-leads-with-yelp---trustpilot-scraping---ai-powered-email-outreach-4912


# Generate Sales Leads with Yelp & Trustpilot Scraping + AI-Powered Email Outreach

### 1. Workflow Overview

This workflow automates the generation of sales leads by scraping business data from Yelp and Trustpilot, analyzing location specifics using AI, and conducting AI-powered personalized email outreach. It targets digital marketing agencies or sales teams seeking to generate qualified leads based on user-defined country, location, and business category. The workflow consists of the following logical blocks:

- **1.1 Input Reception & Location Analysis**: Collect user input on country, city, and business category, then use AI to extract sub-locations within the city.
- **1.2 Yelp Data Scraping & Processing**: Loop through sub-locations to scrape Yelp business data via BrightData APIs, monitor scraping progress, and save results in a Google Sheet.
- **1.3 Yelp Website Extraction & Trustpilot URL Generation**: Extract unique company websites from Yelp data, convert them to Trustpilot review URLs, and remove duplicates.
- **1.4 Trustpilot Data Scraping & Processing**: Loop through Trustpilot URLs, scrape review and company data using BrightData APIs, monitor progress, and save results to another Google Sheet.
- **1.5 Email Extraction & AI Email Generation**: Extract unique emails from Trustpilot data, generate personalized outreach emails using AI, parse the generated content, and send emails via Gmail.
- **1.6 Supporting AI Models**: Use Google Gemini for location analysis and Anthropic Claude for email content generation.
- **1.7 Notes and Documentation**: Provide user guidance and workflow summary via sticky notes.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Location Analysis

- **Overview:** This block starts the workflow by receiving user input through a form and uses AI to analyze the location to extract sub-locations or areas within the city.
- **Nodes Involved:**  
  - Form Trigger - Get User Input  
  - AI Location Analyzer (Langchain Agent)  
  - Split Sub-locations (Code)  

- **Node Details:**

  - **Form Trigger - Get User Input**  
    - Type: Form Trigger (Webhook-based input)  
    - Role: Captures user input fields: country, category, and location (city).  
    - Configuration: Form titled "YelpDataScraper" with three fields: country, category (note trailing space in field name), and location.  
    - Input: External user via webhook  
    - Output: JSON containing input fields  
    - Potential Issues: Missing or malformed input; trailing space on "category " field may cause confusion.

  - **AI Location Analyzer**  
    - Type: Langchain Agent (AI model integration)  
    - Role: Given user input, returns comma-separated sub-locations within the specified city.  
    - Configuration: Prompt instructs AI to list sub-locations based on country, location, and category without extra text or special characters.  
    - Input: Output from form trigger  
    - Output: Text with comma-separated sub-locations  
    - Version-specific: Requires Langchain agent support in n8n; prompt uses template expressions.  
    - Edge Cases: AI may return unexpected formatting; empty or incorrect output; potential API rate limits.

  - **Split Sub-locations**  
    - Type: Code node (JavaScript)  
    - Role: Parses AI output string, cleans it, splits by commas, and outputs each sub-location as an item with associated metadata (category, country).  
    - Key Expressions: Accesses form input via `$` selector, processes AI output string, returns array of objects with id, category, country, and location.  
    - Input: AI Location Analyzer output  
    - Output: Array of structured sub-location objects  
    - Edge Cases: AI output missing or malformed; empty strings; special characters removal.

---

#### 2.2 Yelp Data Scraping & Processing

- **Overview:** For each sub-location, this block scrapes Yelp business data using BrightData API, monitors scraping progress, validates results, fetches data, and stores it in Google Sheets.
- **Nodes Involved:**  
  - Loop Yelp Locations (SplitInBatches)  
  - Yelp Scraper (HTTP Request)  
  - Check Yelp Scrape Progress (HTTP Request)  
  - Wait (1 min) Yelp Completion (Wait)  
  - Verify Yelp Ready (If)  
  - If Yelp Has Records (If)  
  - Fetch Yelp Results (HTTP Request)  
  - Save Yelp Data to Sheet (Google Sheets)  

- **Node Details:**

  - **Loop Yelp Locations**  
    - Type: SplitInBatches  
    - Role: Processes Yelp scraping for one sub-location at a time to prevent overload.  
    - Configuration: Default batch size; processes output from Split Sub-locations node.  
    - Input: Array of sub-locations  
    - Output: Single sub-location per batch  
    - Edge Cases: Large input arrays could slow processing; batch size tuning needed.

  - **Yelp Scraper**  
    - Type: HTTP Request  
    - Role: Triggers BrightData Yelp dataset scraping with specified country, location, and category for each sub-location batch.  
    - Configuration: POST request with JSON body containing input parameters; queries BrightData API with authentication via Bearer token.  
    - Input: Current sub-location batch item  
    - Output: JSON including snapshot_id for tracking progress  
    - Edge Cases: API authentication failures; request timeouts; invalid parameters.

  - **Check Yelp Scrape Progress**  
    - Type: HTTP Request  
    - Role: Polls BrightData API for scraping job progress using snapshot_id.  
    - Configuration: GET request to progress endpoint with Bearer token header.  
    - Input: Snapshot id from Yelp Scraper  
    - Output: JSON status (e.g., "ready") and records count  
    - Edge Cases: Network failures; inconsistent status; API rate limits.

  - **Wait (1 min) Yelp Completion**  
    - Type: Wait  
    - Role: Pauses workflow for 1 minute to allow scraping to complete before checking status again.  
    - Configuration: 1 minute delay  
    - Input: From Check Yelp Scrape Progress  
    - Output: Passes data forward unchanged  
    - Edge Cases: Delay may be insufficient or excessive depending on BrightData responsiveness.

  - **Verify Yelp Ready**  
    - Type: If  
    - Role: Checks if scraping status equals "ready" to proceed.  
    - Configuration: Condition comparing `$json.status` to string "ready".  
    - Input: Check Yelp Scrape Progress output  
    - Output: True branch if ready, else loops back to wait/check.  
    - Edge Cases: Status never reaching ready; infinite loops if not handled.

  - **If Yelp Has Records**  
    - Type: If  
    - Role: Checks if scraped results contain any records (`records` not equal to 0).  
    - Configuration: Number not equals 0 condition on `$json.records`.  
    - Input: From Verify Yelp Ready  
    - Output: True branch fetches results; false branch ends loop for this batch.  
    - Edge Cases: Empty results; API errors returning zero records.

  - **Fetch Yelp Results**  
    - Type: HTTP Request  
    - Role: Retrieves the scraped Yelp data snapshot in JSON format using snapshot_id.  
    - Configuration: GET request to snapshot endpoint with Bearer token and query param `format=json`.  
    - Input: Snapshot id from If Yelp Has Records  
    - Output: JSON array of Yelp business entries  
    - Edge Cases: Snapshot not found; partial data.

  - **Save Yelp Data to Sheet**  
    - Type: Google Sheets  
    - Role: Appends Yelp business data to a specified Google Sheet for lead storage.  
    - Configuration: Maps specific fields from Yelp data to columns (URL, name, address, phone, categories, overall_rating, website); uses Google OAuth2 credentials.  
    - Input: Yelp results JSON array  
    - Output: Confirmation of append operation  
    - Edge Cases: Google API quota limits; mapping errors; invalid sheet ID or permissions.

---

#### 2.3 Yelp Website Extraction & Trustpilot URL Generation

- **Overview:** Extracts unique company websites from the Yelp data sheet, converts them into Trustpilot review URLs, and removes duplicates to prepare for Trustpilot scraping.
- **Nodes Involved:**  
  - Clean Unique Websites (Code)  
  - Read Yelp Sheet Websites (Google Sheets)  
  - Make Trustpilot URLs (Code)  
  - Remove Duplicate TP URLs (Code)  

- **Node Details:**

  - **Clean Unique Websites**  
    - Type: Code  
    - Role: Extracts and deduplicates non-empty company website URLs from input items.  
    - Configuration: Iterates over items, adds trimmed websites to a Set, returns array of unique websites.  
    - Input: Loop Yelp Locations output (Yelp Data saved to sheet)  
    - Output: Items with unique website JSON objects  
    - Edge Cases: Empty or malformed URLs; inconsistent formatting.

  - **Read Yelp Sheet Websites**  
    - Type: Google Sheets  
    - Role: Reads the saved Yelp data sheet to retrieve all company websites.  
    - Configuration: Reads Sheet1 of the given Google Sheet document, using OAuth2 credentials.  
    - Input: Output from Clean Unique Websites  
    - Output: Array of websites read from the sheet  
    - Edge Cases: Sheet access errors; empty sheets.

  - **Make Trustpilot URLs**  
    - Type: Code  
    - Role: Converts company websites into Trustpilot review URLs by cleaning domain and constructing URLs.  
    - Configuration: Uses regex to strip protocol and trailing slash; concatenates with Trustpilot review base URL.  
    - Input: Websites from Google Sheets  
    - Output: Array of Trustpilot website URLs  
    - Edge Cases: Invalid URLs; non-standard domain formats.

  - **Remove Duplicate TP URLs**  
    - Type: Code  
    - Role: Deduplicates Trustpilot URLs to avoid redundant scraping.  
    - Configuration: Uses a Set to track seen URLs and outputs unique items.  
    - Input: Trustpilot URLs from previous node  
    - Output: Unique Trustpilot URLs  
    - Edge Cases: Duplicate URLs; empty input.

---

#### 2.4 Trustpilot Data Scraping & Processing

- **Overview:** Loops through Trustpilot URLs to scrape review and company data, monitors scraping progress, checks data validity, downloads results, and saves them into a Google Sheet.
- **Nodes Involved:**  
  - Loop Trustpilot URLs (SplitInBatches)  
  - Trigger Trustpilot Scraper (HTTP Request)  
  - Check Trustpilot Scrape Progress (HTTP Request)  
  - Verify Trustpilot Scraper Ready (If)  
  - Wait (1 min) Trustpilot Completion (Wait)  
  - If Trustpilot Has Records (If)  
  - Download Trustpilot Data (HTTP Request)  
  - Save Trustpilot Data to Sheet (Google Sheets)  

- **Node Details:**

  - **Loop Trustpilot URLs**  
    - Type: SplitInBatches  
    - Role: Processes Trustpilot scraping one URL at a time.  
    - Configuration: Default batch size, handles unique Trustpilot URLs.  
    - Input: Unique Trustpilot URLs  
    - Output: Single Trustpilot URL item per batch  
    - Edge Cases: Large input size; batch size tuning.

  - **Trigger Trustpilot Scraper**  
    - Type: HTTP Request  
    - Role: Initiates BrightData Trustpilot dataset scraping for each Trustpilot URL.  
    - Configuration: POST request with JSON body including URL and optional date filter; uses Bearer token.  
    - Input: Single Trustpilot URL from batch  
    - Output: Snapshot_id for tracking progress  
    - Edge Cases: API errors; invalid URLs; authentication failures.

  - **Check Trustpilot Scrape Progress**  
    - Type: HTTP Request  
    - Role: Polls status of Trustpilot scraping job by snapshot_id.  
    - Configuration: GET request with Bearer token header, expects JSON response.  
    - Input: Snapshot id from Trigger Trustpilot Scraper  
    - Output: Status (e.g., ready) and records count  
    - Edge Cases: Network errors; API limits.

  - **Verify Trustpilot Scraper Ready**  
    - Type: If  
    - Role: Checks if scraper status is "ready" to continue.  
    - Configuration: Condition checking `$json.status` equals "ready".  
    - Input: Check Trustpilot Scrape Progress output  
    - Output: True branch proceeds; false branch waits again.  
    - Edge Cases: Stuck status; infinite loops.

  - **Wait (1 min) Trustpilot Completion**  
    - Type: Wait  
    - Role: Pauses for 1 minute between progress checks.  
    - Configuration: 1-minute delay  
    - Input: From Verify Trustpilot Scraper Ready false branch  
    - Output: Passes data forward  
    - Edge Cases: Delay inappropriate for actual scraping time.

  - **If Trustpilot Has Records**  
    - Type: If  
    - Role: Validates presence of scraped records by checking if `records` count is not zero.  
    - Configuration: Numeric not equals zero on `$json.records`.  
    - Input: Verify Trustpilot Scraper Ready true branch  
    - Output: True branch downloads data; false branch loops again.  
    - Edge Cases: Missing or empty data.

  - **Download Trustpilot Data**  
    - Type: HTTP Request  
    - Role: Fetches full scraped Trustpilot dataset using snapshot_id.  
    - Configuration: GET request with Bearer token, `format=json` query param.  
    - Input: Snapshot id from If Trustpilot Has Records  
    - Output: JSON array of Trustpilot review and company data  
    - Edge Cases: Partial data; API errors.

  - **Save Trustpilot Data to Sheet**  
    - Type: Google Sheets  
    - Role: Appends scraped Trustpilot data to a designated Google Sheet for email leads.  
    - Configuration: Maps fields like Email, Rating, Address, Company Name, Phone Number, Company About; uses OAuth2 credentials.  
    - Input: Downloaded Trustpilot data JSON  
    - Output: Confirmation of append operation  
    - Edge Cases: Google API limits; missing fields; sheet permissions.

---

#### 2.5 Email Extraction & AI Email Generation

- **Overview:** Extracts unique emails from Trustpilot data, generates personalized outreach emails via AI, parses AI JSON responses, and sends emails through Gmail.
- **Nodes Involved:**  
  - Read Emails from Trustpilot Sheet (Google Sheets)  
  - Get Unique Emails (Code)  
  - AI Generate Email Content (Langchain Agent)  
  - Parse Email JSON (Code)  
  - Send Outreach Email (Gmail)  

- **Node Details:**

  - **Read Emails from Trustpilot Sheet**  
    - Type: Google Sheets  
    - Role: Reads email addresses and related columns from Trustpilot data sheet.  
    - Configuration: Reads specific sheet and document with OAuth2 credentials; includes a filter (example filters on demo@example.com).  
    - Input: Save Trustpilot Data to Sheet output  
    - Output: Sheet rows with email and related info  
    - Edge Cases: Empty or missing emails; filter misconfiguration.

  - **Get Unique Emails**  
    - Type: Code  
    - Role: Deduplicates email addresses from the sheet data.  
    - Configuration: Uses Set to track unique emails, outputs unique email items.  
    - Input: Rows from Google Sheets  
    - Output: Unique email JSON objects  
    - Edge Cases: Empty emails; malformed addresses.

  - **AI Generate Email Content**  
    - Type: Langchain Agent (Claude model)  
    - Role: Generates personalized outreach email content in JSON format for each email.  
    - Configuration: Prompt instructs AI to write a friendly, professional marketing message offering SEO, ads, and website optimization services; outputs JSON containing the email.  
    - Input: Unique email from previous node  
    - Output: Text output containing JSON with email, subject, and message  
    - Edge Cases: AI output not valid JSON; rate limits; content quality.

  - **Parse Email JSON**  
    - Type: Code  
    - Role: Parses AI-generated email JSON output into structured fields (email, subject, content).  
    - Configuration: Removes markdown code blocks if present, attempts JSON parse; fallback regex extraction if parsing fails.  
    - Input: AI Generate Email Content output  
    - Output: Parsed email fields for sending  
    - Edge Cases: Parsing failures; malformed AI output.

  - **Send Outreach Email**  
    - Type: Gmail node  
    - Role: Sends the generated outreach email to the recipient.  
    - Configuration: Uses Gmail OAuth2 credentials; sends to parsed email with generated subject and message content; plain text email.  
    - Input: Parsed email JSON  
    - Output: Confirmation of email sent  
    - Edge Cases: Gmail auth failures; quota limits; invalid email addresses.

---

#### 2.6 Supporting AI Models

- **Gemini - Location AI Model**  
  - Type: Langchain Google Gemini Chat Model  
  - Role: Provides the AI Location Analyzer with sub-location extraction capability.  
  - Configuration: Uses model "models/gemini-1.5-flash", authenticated via Google PaLM API credentials.  
  - Input: From Form Trigger  
  - Output: Feeds AI Location Analyzer node  

- **Claude - Email AI Model**  
  - Type: Langchain Anthropic Claude Chat Model  
  - Role: Powers AI Generate Email Content node to create personalized outreach messages.  
  - Configuration: Uses model "claude-sonnet-4-20250514", authenticated via Anthropic API credentials.  
  - Input: Unique emails  
  - Output: Feeds AI Generate Email Content node  

- **Edge Cases for Both:** API limits, latency, unexpected responses.

---

#### 2.7 Notes and Documentation

- **Sticky Note** (near input trigger):  
  - Content: "Make a Copy of This Google Sheet. (https://docs.google.com/spreadsheets/d/1hX0MD_BLVWuEaXwOjKtwrWsjsBzc32ZtFVjP7wVGQYI/edit?usp=drive_link)"  
  - Role: Instruction for users to prepare their own Google Sheet copy for data storage.

- **Sticky Note1** (overview near top):  
  - Content:  
    ```
    Optimized Workflow Summary:  
    This automation identifies high-quality leads from Yelp and Trustpilot based on a user-submitted location and business category. It uses AI to break down the area into sub-locations, scrapes business details via BrightData, checks credibility through Trustpilot reviews, and stores the best matches in Google Sheets. Finally, AI generates personalized outreach emails, which are automatically sent via Gmail — enabling fully automated lead generation and email marketing with zero manual effort.
    ```  
  - Role: Provides a concise description and summary of workflow purpose and flow.

---

### 3. Summary Table

| Node Name                     | Node Type                        | Functional Role                                    | Input Node(s)                       | Output Node(s)                       | Sticky Note                                                     |
|-------------------------------|---------------------------------|--------------------------------------------------|-----------------------------------|------------------------------------|----------------------------------------------------------------|
| Form Trigger - Get User Input  | Form Trigger                    | Receives user input for country, category, location | None                              | AI Location Analyzer                | Make a Copy of This Google Sheet. (https://docs.google.com/spreadsheets/d/1hX0MD_BLVWuEaXwOjKtwrWsjsBzc32ZtFVjP7wVGQYI/edit?usp=drive_link) |
| AI Location Analyzer           | Langchain Agent (Google Gemini) | Extracts sub-locations from user input             | Form Trigger - Get User Input     | Split Sub-locations                 | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Split Sub-locations            | Code                           | Parses AI output into array of sub-location objects | AI Location Analyzer              | Loop Yelp Locations                | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Loop Yelp Locations            | SplitInBatches                 | Processes each sub-location for Yelp scraping       | Split Sub-locations               | Yelp Scraper, Clean Unique Websites | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Yelp Scraper                  | HTTP Request                   | Triggers Yelp scraping job on BrightData API        | Loop Yelp Locations               | Check Yelp Scrape Progress          | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Check Yelp Scrape Progress     | HTTP Request                   | Checks status of Yelp scraping job                   | Yelp Scraper                     | Wait (1 min) Yelp Completion, Verify Yelp Ready | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Wait (1 min) Yelp Completion   | Wait                           | Waits 1 minute between progress checks               | Check Yelp Scrape Progress        | Verify Yelp Ready                  | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Verify Yelp Ready              | If                             | Verifies if Yelp scraping is complete                | Wait (1 min) Yelp Completion      | If Yelp Has Records, Check Yelp Scrape Progress | Optimized Workflow Summary: This automation identifies high-quality leads... |
| If Yelp Has Records            | If                             | Checks if Yelp scraping returned any records         | Verify Yelp Ready                | Fetch Yelp Results, Loop Yelp Locations | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Fetch Yelp Results             | HTTP Request                   | Downloads Yelp scraped data snapshot                  | If Yelp Has Records              | Save Yelp Data to Sheet            | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Save Yelp Data to Sheet        | Google Sheets                  | Saves Yelp data into Google Sheets                    | Fetch Yelp Results               | Loop Yelp Locations                | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Clean Unique Websites          | Code                           | Extracts and deduplicates company websites           | Loop Yelp Locations              | Read Yelp Sheet Websites           | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Read Yelp Sheet Websites       | Google Sheets                  | Reads websites from saved Yelp data sheet             | Clean Unique Websites            | Make Trustpilot URLs               | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Make Trustpilot URLs           | Code                           | Converts websites to Trustpilot review URLs           | Read Yelp Sheet Websites         | Remove Duplicate TP URLs           | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Remove Duplicate TP URLs       | Code                           | Deduplicates Trustpilot URLs                           | Make Trustpilot URLs             | Loop Trustpilot URLs               | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Loop Trustpilot URLs           | SplitInBatches                 | Processes each Trustpilot URL for scraping            | Remove Duplicate TP URLs         | Trigger Trustpilot Scraper         | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Trigger Trustpilot Scraper     | HTTP Request                   | Initiates Trustpilot scraping job on BrightData API  | Loop Trustpilot URLs             | Check Trustpilot Scrape Progress   | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Check Trustpilot Scrape Progress | HTTP Request                 | Checks status of Trustpilot scraping job              | Trigger Trustpilot Scraper       | Verify Trustpilot Scraper Ready, Wait (1 min) Trustpilot Completion | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Verify Trustpilot Scraper Ready | If                           | Verifies if Trustpilot scraping is complete           | Check Trustpilot Scrape Progress | If Trustpilot Has Records, Wait (1 min) Trustpilot Completion | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Wait (1 min) Trustpilot Completion | Wait                       | Waits 1 minute between Trustpilot scraping progress checks | Verify Trustpilot Scraper Ready | Check Trustpilot Scrape Progress   | Optimized Workflow Summary: This automation identifies high-quality leads... |
| If Trustpilot Has Records      | If                             | Checks if Trustpilot scraping returned records        | Verify Trustpilot Scraper Ready  | Download Trustpilot Data, Loop Trustpilot URLs | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Download Trustpilot Data       | HTTP Request                   | Downloads Trustpilot scraped data snapshot             | If Trustpilot Has Records        | Save Trustpilot Data to Sheet      | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Save Trustpilot Data to Sheet  | Google Sheets                  | Saves Trustpilot data into Google Sheets               | Download Trustpilot Data         | Read Emails from Trustpilot Sheet  | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Read Emails from Trustpilot Sheet | Google Sheets              | Reads emails and data from Trustpilot sheet            | Save Trustpilot Data to Sheet    | Get Unique Emails                  | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Get Unique Emails              | Code                           | Deduplicates email addresses                            | Read Emails from Trustpilot Sheet | AI Generate Email Content          | Optimized Workflow Summary: This automation identifies high-quality leads... |
| AI Generate Email Content      | Langchain Agent (Claude)       | Generates personalized outreach email content          | Get Unique Emails                | Parse Email JSON                  | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Parse Email JSON               | Code                           | Parses AI-generated email JSON output                   | AI Generate Email Content        | Send Outreach Email               | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Send Outreach Email            | Gmail                         | Sends outreach emails via Gmail                          | Parse Email JSON                 | Loop Trustpilot URLs              | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Gemini - Location AI Model     | Langchain LM Chat Google Gemini | Supports AI Location Analyzer with sub-location extraction | Form Trigger - Get User Input (via AI Location Analyzer) | AI Location Analyzer            | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Claude - Email AI Model        | Langchain LM Chat Anthropic    | Supports AI Generate Email Content with outreach messages | Get Unique Emails (via AI Generate Email Content) | AI Generate Email Content        | Optimized Workflow Summary: This automation identifies high-quality leads... |
| Sticky Note                   | Sticky Note                    | User instruction to copy Google Sheet                    | None                            | None                             | Make a Copy of This Google Sheet. (https://docs.google.com/spreadsheets/d/1hX0MD_BLVWuEaXwOjKtwrWsjsBzc32ZtFVjP7wVGQYI/edit?usp=drive_link) |
| Sticky Note1                  | Sticky Note                    | Workflow summary and optimization description             | None                            | None                             | Optimized Workflow Summary: This automation identifies high-quality leads from Yelp and Trustpilot ... |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger - Get User Input**  
   - Type: Form Trigger  
   - Configure form titled "YelpDataScraper" with three fields: "country", "category " (note trailing space), and "location".  
   - Save and note the webhook URL.

2. **Add Gemini - Location AI Model**  
   - Type: Langchain LM Chat Google Gemini  
   - Model: "models/gemini-1.5-flash"  
   - Credentials: Google PaLM API (OAuth2 or API key)  
   - No additional parameters needed.

3. **Add AI Location Analyzer (Langchain Agent)**  
   - Type: Langchain Agent  
   - Prompt: Use the provided prompt template to analyze user input and return comma-separated sub-locations.  
   - Connect Form Trigger to Gemini model (input) and Gemini to AI Location Analyzer (agent node).  
   - Ensure prompt uses expressions to inject form input fields.

4. **Add Split Sub-locations (Code node)**  
   - Input: AI Location Analyzer output  
   - Code: JavaScript that cleans, splits, trims AI output and returns array of sub-location objects with id, category, country, location.  
   - Connect AI Location Analyzer output to this node.

5. **Add Loop Yelp Locations (SplitInBatches)**  
   - Input: Split Sub-locations output  
   - Default batch size; processes one location at a time.

6. **Add Yelp Scraper (HTTP Request)**  
   - POST to `https://api.brightdata.com/datasets/v3/trigger`  
   - Body (JSON): includes country, location, and category from batch item  
   - Query Parameters: `dataset_id=gd_lgugwl0519h1p14rwk`, `include_errors=true`, `type=discover_new`, etc.  
   - Headers: Authorization Bearer token (BrightData API key)  
   - Connect Loop Yelp Locations output as input.

7. **Add Check Yelp Scrape Progress (HTTP Request)**  
   - GET to `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
   - Header: Authorization Bearer token  
   - Connect Yelp Scraper output.

8. **Add Wait (1 min) Yelp Completion (Wait node)**  
   - Wait 1 minute before next check.  
   - Connect Check Yelp Scrape Progress output.

9. **Add Verify Yelp Ready (If node)**  
   - Condition: `$json.status == "ready"`  
   - True branch proceeds; False loops back to Wait.

10. **Add If Yelp Has Records (If node)**  
    - Condition: `$json.records != 0`  
    - True branch fetches results; False ends process for this batch.

11. **Add Fetch Yelp Results (HTTP Request)**  
    - GET to `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}` with `format=json` query param  
    - Header: Authorization Bearer token  
    - Connect from If Yelp Has Records true branch.

12. **Add Save Yelp Data to Sheet (Google Sheets)**  
    - Operation: Append  
    - Document ID: Your Google Sheet ID for Yelp data  
    - Sheet Name: e.g., "Sheet1" or `gid=0`  
    - Map columns: URL, name, address, phone, categories, overall_rating, website  
    - Credentials: Google Sheets OAuth2  
    - Connect Fetch Yelp Results output.

13. **Add Clean Unique Websites (Code node)**  
    - Extract and deduplicate "Company Website" from saved Yelp data.  
    - Connect Loop Yelp Locations output to this node.

14. **Add Read Yelp Sheet Websites (Google Sheets)**  
    - Reads the Yelp data sheet to get websites.  
    - Document ID and Sheet Name same as Save Yelp Data to Sheet.  
    - Credentials: Google OAuth2  
    - Connect Clean Unique Websites output.

15. **Add Make Trustpilot URLs (Code node)**  
    - Convert each website to Trustpilot review URL by cleaning domain and appending to `https://www.trustpilot.com/review/`.  
    - Connect Read Yelp Sheet Websites output.

16. **Add Remove Duplicate TP URLs (Code node)**  
    - Deduplicate Trustpilot URLs using Set.  
    - Connect Make Trustpilot URLs output.

17. **Add Loop Trustpilot URLs (SplitInBatches)**  
    - Batch process Trustpilot URLs for scraping.  
    - Connect Remove Duplicate TP URLs output.

18. **Add Trigger Trustpilot Scraper (HTTP Request)**  
    - POST to `https://api.brightdata.com/datasets/v3/trigger` with Trustpilot URL input  
    - Dataset ID: `gd_lm5zmhwd2sni130p`  
    - Header: Authorization Bearer token  
    - Connect Loop Trustpilot URLs output.

19. **Add Check Trustpilot Scrape Progress (HTTP Request)**  
    - GET to `https://api.brightdata.com/datasets/v3/progress/{{ $json.snapshot_id }}`  
    - Header: Authorization Bearer token  
    - Connect Trigger Trustpilot Scraper output.

20. **Add Verify Trustpilot Scraper Ready (If node)**  
    - Condition: `$json.status == "ready"`  
    - True branch proceeds; False branch waits.

21. **Add Wait (1 min) Trustpilot Completion (Wait node)**  
    - 1-minute delay  
    - Connect Verify Trustpilot Scraper Ready false branch.

22. **Add If Trustpilot Has Records (If node)**  
    - Condition: `$json.records != 0`  
    - True branch downloads data; False loops again.

23. **Add Download Trustpilot Data (HTTP Request)**  
    - GET to `https://api.brightdata.com/datasets/v3/snapshot/{{ $json.snapshot_id }}?format=json`  
    - Header: Authorization Bearer token  
    - Connect If Trustpilot Has Records true branch.

24. **Add Save Trustpilot Data to Sheet (Google Sheets)**  
    - Append mode  
    - Document ID: Your Google Sheet ID for Trustpilot data  
    - Sheet Name: e.g., "Mail Scrap" or specific gid  
    - Map columns: Email, Rating, Address, Company Name, Phone Number, Company About  
    - Google Sheets OAuth2 credentials  
    - Connect Download Trustpilot Data output.

25. **Add Read Emails from Trustpilot Sheet (Google Sheets)**  
    - Reads emails from Trustpilot data sheet  
    - Optional filter example on Email column (can be omitted or customized)  
    - Connect Save Trustpilot Data to Sheet output.

26. **Add Get Unique Emails (Code node)**  
    - Deduplicates email addresses from sheet data.  
    - Connect Read Emails from Trustpilot Sheet output.

27. **Add Claude - Email AI Model (Langchain Agent)**  
    - Model: "claude-sonnet-4-20250514"  
    - Credentials: Anthropic API key  
    - Connect Get Unique Emails output.

28. **Add AI Generate Email Content (Langchain Agent)**  
    - Prompt instructs AI to write a professional marketing outreach email offering SEO, ads, and website optimization.  
    - Output requested in JSON format containing "email".  
    - Connect Claude Email AI Model output.

29. **Add Parse Email JSON (Code node)**  
    - Parses JSON from AI output, cleaning code block markdown if present, extracting email, subject, and content fields.  
    - Connect AI Generate Email Content output.

30. **Add Send Outreach Email (Gmail node)**  
    - Sends email using Gmail OAuth2 credentials  
    - Recipient: Parsed email  
    - Subject and message: parsed content from previous node  
    - Connect Parse Email JSON output.

31. **Connect Send Outreach Email output back to Loop Trustpilot URLs**  
    - Allows processing next email batch or ends flow.

32. **Add Sticky Notes**  
    - Add user instructions and workflow summary for clarity.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                           |
|------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Make a Copy of This Google Sheet. (https://docs.google.com/spreadsheets/d/1hX0MD_BLVWuEaXwOjKtwrWsjsBzc32ZtFVjP7wVGQYI/edit?usp=drive_link) | Instruction to prepare a Google Sheet for storing scraped data and leads.                                |
| Optimized Workflow Summary: This automation identifies high-quality leads from Yelp and Trustpilot based on user input, uses AI to extract sub-locations, scrapes business and review data, stores in Google Sheets, generates personalized outreach emails, and sends them automatically via Gmail. | Workflow purpose and high-level process description.                                                     |

---

**Disclaimer:** The provided text and workflow come exclusively from an n8n automated workflow. This respects all applicable content policies and contains no illegal or protected content. All data processed is legal and public.