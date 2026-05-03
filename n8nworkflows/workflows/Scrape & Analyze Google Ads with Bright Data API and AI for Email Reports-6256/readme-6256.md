Scrape & Analyze Google Ads with Bright Data API and AI for Email Reports

https://n8nworkflows.xyz/workflows/scrape---analyze-google-ads-with-bright-data-api-and-ai-for-email-reports-6256


# Scrape & Analyze Google Ads with Bright Data API and AI for Email Reports

### 1. Workflow Overview

This workflow automates the process of scraping Google Ads data via the Bright Data API, analyzing the ad content using AI models, and generating structured email reports summarizing key insights. Its primary use case is for marketing analysts or strategists who want to track keyword-triggered Google Ads, understand advertiser messaging strategies, and receive summarized performance reports via email.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & Keyword Retrieval:** Trigger and fetch list of keywords from Google Sheets.
- **1.2 Google Ads Scraping:** For each keyword and country code, scrape Google Search results JSON including ads via Bright Data API.
- **1.3 Ads Filtering & Extraction:** Filter results to ensure ads exist, then extract top and bottom ads arrays.
- **1.4 AI Analysis of Ads:** Use multiple AI language models to analyze ads for value propositions, messaging style, and extensions.
- **1.5 Site Link Extensions Analysis:** Extract and categorize site link extensions from ads.
- **1.6 Keyword-to-Ad Mapping:** Map keywords to ads and analyze advertiser reuse and messaging differences.
- **1.7 Report Generation & Emailing:** Combine AI insights, generate an HTML report, and send it via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Keyword Retrieval

- **Overview:**  
Starts the workflow on manual trigger and retrieves a list of keywords and country codes from a Google Sheet to be processed.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Get Keywords (Google Sheets)  
  - Filter1 (Filter)

- **Node Details:**  
  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Connections: Outputs to Get Keywords.  
    - Edge cases: No major failure expected.  
  - **Get Keywords**  
    - Type: Google Sheets  
    - Role: Reads keywords and country codes from a specified Google Sheet tab.  
    - Config: Document ID and sheet GID set to a shared public Google Sheet.  
    - Credentials: Uses Google Sheets OAuth2 credentials.  
    - Outputs: Rows with columns including `Keyword` and `country code`.  
    - Edge cases: OAuth token expiration, empty sheet, or connectivity issues.  
  - **Filter1**  
    - Type: Filter  
    - Role: Filters out any records with empty keyword field.  
    - Config: Condition to pass only if `Keyword` is not empty.  
    - Connections: Passes filtered keywords to Loop Over Items.  
    - Edge cases: No keywords pass filter resulting in empty processing.

---

#### 2.2 Google Ads Scraping

- **Overview:**  
Loops over each keyword/country code, submits scraping requests to Bright Data API to fetch Google Search JSON results including ads.

- **Nodes Involved:**  
  - Loop Over Items (Splitting batch to single items)  
  - set keyword (Set)  
  - Fetch Google Search Results JSON (HTTP Request)  
  - filter ads (Filter)  
  - ads found (Set)  
  - No Operation, do nothing1 (NoOp)  

- **Node Details:**  
  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes one keyword item at a time to handle rate limits and sequential scraping.  
    - Edge cases: Batch size or rate limits causing delays.  
  - **set keyword**  
    - Type: Set  
    - Role: Formats the current item to create `search_term` and `country code` variables used in scraping request.  
    - Key Expressions: Sets `search_term` = Keyword, `country code` as is.  
  - **Fetch Google Search Results JSON**  
    - Type: HTTP Request  
    - Role: Calls Bright Data API with parameters for zone (serp_api1), Google search URL with query, country, and format.  
    - Config: POST request, async param true, uses HTTP header authentication with Bright Data credentials.  
    - Expressions: URL dynamically built with `search_term` and `country code`.  
    - Edge cases: API errors, rate limits, malformed requests or auth errors. Has error continuation configured.  
  - **filter ads**  
    - Type: Filter  
    - Role: Checks if `top_ads` array from the JSON response has at least one element to proceed.  
    - Edge cases: No ads found causing filter to skip.  
  - **ads found**  
    - Type: Set  
    - Role: Extracts `top_ads` and `bottom_ads` arrays from the response JSON for downstream processing.  
  - **No Operation, do nothing1**  
    - Type: NoOp  
    - Role: Acts as a placeholder for the error branch of the HTTP request.

---

#### 2.3 Ads Filtering & Extraction

- **Overview:**  
Extracts individual ads from the combined top and bottom ads arrays for detailed analysis and writes raw ad data to Google Sheets.

- **Nodes Involved:**  
  - extract ads (Code)  
  - extensions (Aggregate)  
  - value proposition1 (Aggregate)  
  - keywords (Aggregate)  
  - Split ads (SplitOut)  
  - Edit Fields (Set)  
  - Top ads (Google Sheets)

- **Node Details:**  
  - **extract ads**  
    - Type: Code  
    - Role: Combines `top_ads` and `bottom_ads` arrays and outputs each ad as separate workflow items for parallel processing.  
    - Code: Maps combined ads array to individual JSON items.  
  - **extensions, value proposition1, keywords**  
    - Type: Aggregate  
    - Role: Aggregates all ad items' data except excluded fields to prepare batches for AI analysis steps.  
  - **Split ads**  
    - Type: SplitOut  
    - Role: Splits combined ad arrays field into individual items for Google Sheets insertion.  
  - **Edit Fields**  
    - Type: Set  
    - Role: Adds a static `keyword` field ("Business automations") for context in the Google Sheet.  
  - **Top ads**  
    - Type: Google Sheets  
    - Role: Appends cleaned and formatted top ads data to a specific Google Sheet tab for records.  
    - Credentials: Google Sheets OAuth2 account.  
    - Edge cases: Sheet access or quota limitations.

---

#### 2.4 AI Analysis of Ads

- **Overview:**  
Runs multiple AI language models to analyze ad messaging, value propositions, and extract structured insights.

- **Nodes Involved:**  
  - Value Proposition & Messaging Analysis (Chain LLM)  
  - Site Link Extension Mining (Chain LLM)  
  - Keyword-Ad Mapping (Chain LLM)  
  - OpenRouter Chat Model (LM Chat)  
  - Google Gemini Chat Model (LM Chat)  
  - Structured Output Parser nodes (Output Parsers)  
  - value proposition (Set)  
  - site links (Set)  
  - keywords1 (Set)

- **Node Details:**  
  - **Value Proposition & Messaging Analysis**  
    - Type: Chain LLM  
    - Role: Analyzes ads' JSON data to identify core value propositions, tone, marketing devices, and common phrases.  
    - Prompt: Detailed instructions to classify emotional hooks, rational benefits, guarantees, and marketing claims.  
    - Output: Structured JSON with advertiser insights.  
  - **Site Link Extension Mining**  
    - Type: Chain LLM  
    - Role: Extracts and categorizes site link extensions and CTA strategies from ad extensions JSON.  
    - Output: Structured JSON listing site link text, URLs, types, and UTM presence.  
  - **Keyword-Ad Mapping**  
    - Type: Chain LLM  
    - Role: Maps keywords to ads, identifies advertiser reuse, clusters related keywords, and compares messaging differences.  
    - Output: Structured JSON summarizing keyword-ad relations and competitive insights.  
  - **OpenRouter Chat Model & Google Gemini Chat Model**  
    - Type: Language Model Chat nodes  
    - Role: Serve as AI engines for the chain LLM nodes, providing alternative AI model endpoints (OpenRouter, Google Gemini).  
    - Credentials: Requires respective API credentials.  
  - **Structured Output Parsers**  
    - Type: Output Parser  
    - Role: Parses AI output to ensure it conforms to the expected JSON schema.  
  - **value proposition, site links, keywords1**  
    - Type: Set  
    - Role: Stores AI output from each analysis step into named JSON arrays for merging later.  

---

#### 2.5 Report Generation & Emailing

- **Overview:**  
Merges all AI insights and generates a detailed, structured HTML email report sent to a specified recipient.

- **Nodes Involved:**  
  - Merge1 (Merge)  
  - Generate HTML (Agent LLM)  
  - Send a message (Gmail)

- **Node Details:**  
  - **Merge1**  
    - Type: Merge  
    - Role: Combines AI insights for value proposition, site links, and keyword-ad mapping by position to produce a unified JSON input for report generation.  
  - **Generate HTML**  
    - Type: Agent LLM  
    - Role: Uses the merged JSON data to generate a professional HTML-formatted email body summarizing all insights.  
    - Prompt: Specifies formatting with HTML tags `<h2>`, `<ul>`, `<p>`, `<strong>`, grouping by advertiser, and focusing on clarity for strategists.  
  - **Send a message**  
    - Type: Gmail  
    - Role: Sends the generated HTML report via Gmail to a fixed email address with a subject referencing the keyword.  
    - Credentials: Gmail OAuth2 account.  
    - Edge cases: Email sending failure, OAuth expiration.

---

### 3. Summary Table

| Node Name                    | Node Type                            | Functional Role                          | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                                                                                                           |
|------------------------------|------------------------------------|----------------------------------------|---------------------------------|--------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                     | Entry point to start the workflow      |                                 | Get Keywords                   |                                                                                                                                                                                       |
| Get Keywords                 | Google Sheets                      | Fetch keywords and country codes       | When clicking ‘Execute workflow’ | Filter1                       | - Make a copy of this [G sheet](https://docs.google.com/spreadsheets/d/1QU9rwawCZLiYW8nlYYRMj-9OvAUNZoe2gP49KbozQqw/edit?usp=sharing) - Add your desired keywords                       |
| Filter1                     | Filter                            | Filter non-empty keywords               | Get Keywords                    | Loop Over Items                | - We loop over each item one at a time                                                                                                                                               |
| Loop Over Items             | SplitInBatches                    | Process keywords one by one             | Filter1                        | set keyword                   | - Map keyword and country code - Update the Zone name to match your zone on Bright Data - Run the scraper                                                                             |
| set keyword                 | Set                              | Prepare search_term and country code   | Loop Over Items                | Fetch Google Search Results JSON |                                                                                                                                                                                       |
| Fetch Google Search Results JSON | HTTP Request                     | Scrape Google Ads JSON via Bright Data | set keyword                   | filter ads, No Operation, do nothing1 | ## Setting up SERP scraper in Bright Data 1. On Bright Data, go to the [Proxies & Scraping](https://brightdata.com/cp/zones) tab 2. Under SERP API, create a new zone 3. Add this to account |
| filter ads                 | Filter                            | Check if ads are present                | Fetch Google Search Results JSON | ads found                     |                                                                                                                                                                                       |
| No Operation, do nothing1  | NoOp                             | Handle error branch                     | Fetch Google Search Results JSON |                               |                                                                                                                                                                                       |
| ads found                  | Set                              | Extract top_ads and bottom_ads arrays  | filter ads                     | extract ads                   |                                                                                                                                                                                       |
| extract ads                | Code                             | Split combined ads into individual items | ads found                    | value proposition1, extensions, keywords, Split ads |                                                                                                                                                                                       |
| value proposition1         | Aggregate                        | Aggregate ad data for AI analysis      | extract ads                   | Value Proposition & Messaging Analysis |                                                                                                                                                                                       |
| extensions                 | Aggregate                        | Aggregate ad extensions data            | extract ads                   | Site Link Extension Mining    |                                                                                                                                                                                       |
| keywords                   | Aggregate                        | Aggregate ad data for keyword mapping  | extract ads                   | Keyword-Ad Mapping            |                                                                                                                                                                                       |
| Split ads                 | SplitOut                         | Split ads array for Google Sheets      | extract ads                   | Edit Fields                  |                                                                                                                                                                                       |
| Edit Fields               | Set                              | Add keyword field for Google Sheets    | Split ads                    | Top ads                     |                                                                                                                                                                                       |
| Top ads                   | Google Sheets                    | Append top ads data to Google Sheets   | Edit Fields                  |                              |                                                                                                                                                                                       |
| Value Proposition & Messaging Analysis | Chain LLM                        | AI analysis of ad messaging             | value proposition1            | value proposition            | ## Extract the value proposition and message                                                                                                                                        |
| Site Link Extension Mining | Chain LLM                        | AI analysis of ad extensions            | extensions                   | site links                  | ## Extract Insights from Sitelinks ad extensions                                                                                                                                    |
| Keyword-Ad Mapping        | Chain LLM                        | AI analysis mapping keywords to ads    | keywords                     | keywords1                   | ## Keyword mapping                                                                                                                                                                    |
| value proposition          | Set                              | Store AI output for value proposition  | Value Proposition & Messaging Analysis | Merge1                      |                                                                                                                                                                                       |
| site links                | Set                              | Store AI output for site link analysis | Site Link Extension Mining   | Merge1                      |                                                                                                                                                                                       |
| keywords1                 | Set                              | Store AI output for keyword-ad mapping | Keyword-Ad Mapping           | Merge1                      |                                                                                                                                                                                       |
| Merge1                    | Merge                            | Combine AI insights for report         | value proposition, site links, keywords1 | Generate HTML              |                                                                                                                                                                                       |
| Generate HTML             | Agent LLM                        | Generate HTML email report              | Merge1                       | Send a message              | ## Generate HTML from insights                                                                                                                                                       |
| Send a message            | Gmail                           | Send report email                       | Generate HTML                |                              |                                                                                                                                                                                       |
| Sticky Note4               | Sticky Note                     | Instructions for setting up keywords   |                               |                              | - Make a copy of this [G sheet](https://docs.google.com/spreadsheets/d/1QU9rwawCZLiYW8nlYYRMj-9OvAUNZoe2gP49KbozQqw/edit?usp=sharing) - Add your desired keywords                       |
| Sticky Note5               | Sticky Note                     | Explanation of looping over items      |                               |                              | - We loop over each item one at a time                                                                                                                                               |
| Sticky Note6               | Sticky Note                     | Reminder to update zone name and run scraper |                          |                              | - Map keyword and country code - Update the Zone name to match your zone on Bright Data - Run the scraper                                                                             |
| Sticky Note7               | Sticky Note                     | Setup instructions for Bright Data     |                               |                              | ## Setting up SERP scraper in Bright Data 1. On Bright Data, go to the [Proxies & Scraping](https://brightdata.com/cp/zones) tab 2. Under SERP API, create a new zone 3. Add this to account |
| Sticky Note12              | Sticky Note                     | Overview of AI processing of top ads   |                               |                              | ## AI processing of the top ads These AI steps analyze and extract different results from the top and bottom ads found on the given page per keyword                                |
| Sticky Note13              | Sticky Note                     | Label for Generate HTML step            |                               |                              | ## Generate HTML from insights                                                                                                                                                       |
| Sticky Note14              | Sticky Note                     | Label for value proposition extraction |                               |                              | ## Extract the value proposition and message                                                                                                                                        |
| Sticky Note15              | Sticky Note                     | Label for site link extraction          |                               |                              | ## Extract Insights from Sitelinks ad extensions                                                                                                                                    |
| Sticky Note16              | Sticky Note                     | Label for keyword mapping                |                               |                              | ## Keyword mapping                                                                                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Name: When clicking ‘Execute workflow’  
   - Type: Manual Trigger  
   - No parameters.

2. **Add Google Sheets Node to Get Keywords:**  
   - Name: Get Keywords  
   - Type: Google Sheets  
   - Configure with your Google Sheets OAuth2 credentials.  
   - Document ID: Use your keywords tracking sheet ID.  
   - Sheet Name: Use the GID or tab name containing keywords and country codes.  
   - No filters.  
   - Connect output from Manual Trigger.

3. **Add Filter Node to Exclude Empty Keywords:**  
   - Name: Filter1  
   - Type: Filter  
   - Condition: `Keyword` field is not empty (string not empty).  
   - Connect input from Get Keywords.

4. **Add SplitInBatches Node to Loop Over Keywords:**  
   - Name: Loop Over Items  
   - Type: SplitInBatches  
   - Batch Size: default (1) to process items one by one.  
   - Connect input from Filter1.

5. **Add Set Node to Prepare API Request Variables:**  
   - Name: set keyword  
   - Type: Set  
   - Assign `search_term` with `{{$json.Keyword}}`  
   - Assign `country code` with `{{$json['country code']}}`  
   - Connect input from Loop Over Items (main output).

6. **Add HTTP Request Node to Call Bright Data API:**  
   - Name: Fetch Google Search Results JSON  
   - Type: HTTP Request (POST)  
   - URL: `https://api.brightdata.com/request`  
   - Authentication: HTTP Header Auth with Bright Data API key.  
   - Body Parameters:  
     - zone: your zone name (e.g., serp_api1)  
     - url: `https://www.google.com/search?q={{$json.search_term.replaceAll(" ", "+")}}&start=0&brd_json=1`  
     - country: `{{$json["country code"]}}`  
     - format: raw  
   - Query Parameters: `async=true`  
   - Error Strategy: Continue on error (to handle API failures gracefully).  
   - Connect input from set keyword.

7. **Add Filter Node to Ensure Ads Present:**  
   - Name: filter ads  
   - Type: Filter  
   - Condition: `top_ads` array length >= 1.  
   - Connect input from HTTP Request (success output).

8. **Add Set Node to Extract Ads Arrays:**  
   - Name: ads found  
   - Type: Set  
   - Assign `top_ads` and `bottom_ads` fields from response JSON.  
   - Connect input from filter ads.

9. **Add Code Node to Split Ads into Individual Items:**  
   - Name: extract ads  
   - Type: Code  
   - Code to combine `top_ads` and `bottom_ads`, output each ad as separate item.  
   - Connect input from ads found.

10. **Add Aggregate Nodes to Prepare Data for AI Analysis:**  
    - Name: value proposition1 (aggregate all ad fields except referral_link, image, etc.)  
    - Name: extensions (aggregate all ad fields except some image/rank fields)  
    - Name: keywords (aggregate all ad fields except image/rank fields)  
    - Connect all from extract ads.

11. **Add SplitOut Node to Split Ads for Google Sheets:**  
    - Name: Split ads  
    - Type: SplitOut  
    - Field to split: `bottom_ads, top_ads`  
    - Connect input from extract ads.

12. **Add Set Node to Add Keyword Field:**  
    - Name: Edit Fields  
    - Type: Set  
    - Assign static `keyword` field (e.g., "Business automations").  
    - Connect input from Split ads.

13. **Add Google Sheets Node to Append Top Ads:**  
    - Name: Top ads  
    - Type: Google Sheets  
    - Configure with Google Sheets OAuth2 credentials.  
    - Document ID and sheet tab for top ads.  
    - Append mode with auto column mapping.  
    - Connect input from Edit Fields.

14. **Add Chain LLM Nodes for AI Analysis:**  
    - Value Proposition & Messaging Analysis: Provide prompt to analyze ad messaging and value prop.  
    - Site Link Extension Mining: Provide prompt to extract and categorize site link extensions.  
    - Keyword-Ad Mapping: Provide prompt to map keywords to ads and analyze competitive differences.  
    - Connect inputs respectively from value proposition1, extensions, keywords aggregates.

15. **Add Language Model Chat Nodes (OpenRouter, Google Gemini) for AI Engines:**  
    - Configure with respective API credentials.  
    - Connect these as the AI model nodes for each Chain LLM node accordingly.

16. **Add Structured Output Parser Nodes:**  
    - For each AI analysis Chain LLM node, add a structured output parser to validate JSON.  
    - Connect output of each Chain LLM to its parser.

17. **Add Set Nodes to Store AI Outputs:**  
    - Name: value proposition, site links, keywords1  
    - Assign output JSON of respective AI analyses to named variables.  
    - Connect from respective structured output parsers.

18. **Add Merge Node to Combine AI Insights:**  
    - Name: Merge1  
    - Mode: Combine by position (combine 3 inputs).  
    - Connect inputs from value proposition, site links, keywords1 Set nodes.

19. **Add Agent LLM Node to Generate HTML Report:**  
    - Name: Generate HTML  
    - Provide prompt to generate an HTML formatted marketing report from combined JSON insights.  
    - Connect input from Merge1.

20. **Add Gmail Node to Send Email:**  
    - Name: Send a message  
    - Configure with Gmail OAuth2 credentials.  
    - Set recipient email address.  
    - Subject includes keyword dynamically: `Ads overview report for the Keyword [{{ $('set keyword').item.json.search_term }}]`  
    - Message body from Generate HTML output.  
    - Connect input from Generate HTML.

21. **Connect branches for error handling and no ads found:**  
    - For HTTP request errors, connect to a No Operation node to safely skip.

22. **Add Sticky Notes for Documentation:**  
    - Add notes near key nodes describing their purpose and setup instructions, including links to Google Sheets and Bright Data setup.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Make a copy of this [Google Sheet](https://docs.google.com/spreadsheets/d/1QU9rwawCZLiYW8nlYYRMj-9OvAUNZoe2gP49KbozQqw/edit?usp=sharing) and add your desired keywords | Keyword input source                                                                                             |
| Setup instructions for Bright Data SERP scraper: go to [Bright Data Zones](https://brightdata.com/cp/zones), create a SERP API zone, and use that zone name in the workflow | Bright Data API setup                                                                                            |
| AI analysis prompts cover advertising psychology, value proposition extraction, CTA strategy, and keyword-ad mapping | For understanding AI-driven insights                                                                             |
| The final email report is generated in HTML format with structured sections and sent via Gmail to the specified address | Use for automated marketing performance reporting                                                                |

---

This reference document enables you to understand, reproduce, modify, and troubleshoot the entire workflow that scrapes Google Ads data, analyzes it with AI, and sends insightful email reports.