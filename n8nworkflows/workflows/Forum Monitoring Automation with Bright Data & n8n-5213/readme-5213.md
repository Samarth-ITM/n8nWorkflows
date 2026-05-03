Forum Monitoring Automation with Bright Data & n8n

https://n8nworkflows.xyz/workflows/forum-monitoring-automation-with-bright-data---n8n-5213


# Forum Monitoring Automation with Bright Data & n8n

---
### 1. Workflow Overview

This workflow automates monitoring online forum discussions specifically on Quora related to a targeted keyword or product (e.g., "iPhone 16"). It uses Bright Data’s proxy-powered scraping capabilities to bypass anti-bot restrictions and gather fresh data periodically. The workflow is structured into four main logical blocks:

- **1.1 Define and Trigger the Search:** Sets the periodic trigger and defines the search keyword to monitor.
- **1.2 Scrape Quora Discussion Links via Google:** Performs a Google search filtered to Quora, extracts thread links, and formats them.
- **1.3 Scrape Actual Quora Answers:** Fetches full HTML of each Quora thread and extracts user answers.
- **1.4 Summarize & Store the Insights:** Uses OpenAI to generate summarized feedback from extracted answers and saves results into Google Sheets.

This modular design allows flexible topic monitoring, leveraging proxy scraping, AI summarization, and cloud storage for insights.

---

### 2. Block-by-Block Analysis

#### 2.1 Define and Trigger the Search

- **Overview:**  
  This block initializes the automation by triggering it on a schedule and provides the keyword/topic to search for.

- **Nodes Involved:**  
  - Run Periodically  
  - Define Search Keyword

- **Node Details:**

  1. **Run Periodically**  
     - Type: Schedule Trigger  
     - Role: Starts the workflow every day at 09:00 AM automatically.  
     - Configuration: Set with a daily interval triggering at hour 9.  
     - Inputs: None (trigger node)  
     - Outputs: Connects to "Define Search Keyword"  
     - Edge Cases: Ensure that n8n instance timezone matches expected schedule; failure to trigger if scheduler halted.

  2. **Define Search Keyword**  
     - Type: Set  
     - Role: Defines the string keyword to be searched ("iPhone+16" by default).  
     - Configuration: Static string assignment to variable `keyword`.  
     - Inputs: From "Run Periodically"  
     - Outputs: Connects to "Google Search Results"  
     - Edge Cases: Keyword should be URL-encoded or sanitized for Google search; empty or invalid keywords will produce no results.

---

#### 2.2 Scrape Quora Discussion Links via Google

- **Overview:**  
  Performs a Google search restricted to Quora, extracts thread titles and URLs, and formats these URLs for further processing.

- **Nodes Involved:**  
  - Google Search Results  
  - Links from Google HTML  
  - Format Thread URLs

- **Node Details:**

  1. **Google Search Results**  
     - Type: HTTP Request  
     - Role: Sends a POST request to Bright Data’s API to perform a Google search for `site:quora.com+{{keyword}}`.  
     - Configuration:  
       - URL: Bright Data API endpoint `https://api.brightdata.com/request`  
       - Method: POST, JSON body includes proxy zone, target URL, country US, raw HTML format, and user-agent header.  
       - Authorization: Bearer token in header (replace `API_KEY` with actual key).  
     - Inputs: From "Define Search Keyword"  
     - Outputs: Connects to "Links from Google HTML"  
     - Edge Cases:  
       - Bright Data API key missing or invalid -> auth failure.  
       - Rate limits or quota exceeded on Bright Data side.  
       - Google may return CAPTCHA or no results if query malformed.

  2. **Links from Google HTML**  
     - Type: HTML Extract  
     - Role: Parses the raw HTML from Google search results to extract:  
       - Thread Titles (from `<h3>`)  
       - URLs (from `div.yuRUbf a` links' href attributes)  
     - Configuration: CSS selectors for extraction, returns arrays for titles and URLs.  
     - Inputs: From "Google Search Results"  
     - Outputs: Connects to "Format Thread URLs"  
     - Edge Cases: If Google changes page structure, selectors may break extraction.

  3. **Format Thread URLs**  
     - Type: Code (JavaScript)  
     - Role: Matches extracted titles and URLs into an array of objects `{title, url}` to normalize output.  
     - Configuration: Custom JavaScript mapping over extracted arrays, only pairs up to minimum length of titles or URLs.  
     - Inputs: From "Links from Google HTML"  
     - Outputs: Connects to "Individual Quora Threads"  
     - Edge Cases: Mismatch in lengths of titles and URLs may cause incomplete pairs; empty arrays produce no output.

---

#### 2.3 Scrape Actual Quora Answers

- **Overview:**  
  Visits each Quora thread URL obtained previously, scrapes full HTML content, and extracts relevant user answers from the page.

- **Nodes Involved:**  
  - Individual Quora Threads  
  - Answers from Threads

- **Node Details:**

  1. **Individual Quora Threads**  
     - Type: HTTP Request  
     - Role: Uses Bright Data API again to fetch raw HTML content of each individual Quora thread URL.  
     - Configuration:  
       - POST request to Bright Data API  
       - JSON body dynamically sets `url` from each thread’s `url` field  
       - Same headers and authorization approach as Google Search Results.  
     - Inputs: From "Format Thread URLs"  
     - Outputs: Connects to "Answers from Threads"  
     - Edge Cases:  
       - Errors if Bright Data API key invalid or usage exceeded.  
       - Quora page structure changes may impact scraping.  
       - Network timeouts or blocked IPs.

  2. **Answers from Threads**  
     - Type: HTML Extract  
     - Role: Extracts answers or user comments from each thread’s HTML using CSS selectors.  
     - Configuration: Uses a specific CSS selector targeting Quora answers (note: selector references a sample HTML snippet with styled span).  
     - Inputs: From "Individual Quora Threads"  
     - Outputs: Connects to "AI Feedback Summary1"  
     - Edge Cases: Selector may fail if Quora changes page layout or answer formatting; empty results if thread has no answers.

---

#### 2.4 Summarize & Store the Insights

- **Overview:**  
  Aggregates extracted answers, feeds them into an OpenAI model to generate a concise summary, and saves the summary along with the topic keyword into a Google Sheet.

- **Nodes Involved:**  
  - AI Feedback Summary1  
  - OpenAI Chat Model (connected internally by LangChain agent)  
  - Save to Google Sheet

- **Node Details:**

  1. **AI Feedback Summary1**  
     - Type: LangChain Agent (AI processing)  
     - Role: Summarizes the consumer feedback text extracted from Quora answers using an OpenAI language model.  
     - Configuration:  
       - Input text template: `"Summarize the following consumer feedback from Quora:\n{{ $json.answers }}"`  
       - Uses OpenAI credentials for API calls.  
     - Inputs: From "Answers from Threads"  
     - Outputs: Connects to "Save to Google Sheet"  
     - Edge Cases:  
       - OpenAI API quota or key issues may cause errors.  
       - Long input text might exceed token limits; may require chunking in extended workflows.

  2. **OpenAI Chat Model**  
     - Type: LangChain LLM Node (GPT-4o-mini)  
     - Role: Backend model invoked by the agent to generate the summary.  
     - Configuration: Model set to "gpt-4o-mini".  
     - Inputs: Connected internally by LangChain from "AI Feedback Summary1"  
     - Outputs: Returns AI-generated summary to agent node.  
     - Edge Cases: Model availability or version changes might affect results.

  3. **Save to Google Sheet**  
     - Type: Google Sheets Node  
     - Role: Appends a row to a specific Google Sheet with columns: Topic and Discussion Summary.  
     - Configuration:  
       - Document ID: linked to a specific Google Sheet (provided)  
       - Sheet name: set to first sheet (`gid=0`)  
       - Columns mapped:  
         - Topic → keyword from "Define Search Keyword" node  
         - Discussion Summary → AI-generated summary text  
       - Google Sheets OAuth2 credentials configured for access.  
     - Inputs: From "AI Feedback Summary1"  
     - Outputs: None (end of workflow)  
     - Edge Cases: Authentication failures, permission issues, or sheet limits may cause failures.

---

### 3. Summary Table

| Node Name              | Node Type                              | Functional Role                               | Input Node(s)         | Output Node(s)            | Sticky Note                                                                                                                                                                                                                                  |
|------------------------|--------------------------------------|-----------------------------------------------|-----------------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Run Periodically        | Schedule Trigger                     | Triggers workflow daily at 9 AM                | None                  | Define Search Keyword      | Section 1: Define and Trigger the Search - Starts workflow automatically on schedule                                                                                                                                                         |
| Define Search Keyword   | Set                                 | Defines search keyword (e.g., "iPhone 16")    | Run Periodically       | Google Search Results      | Section 1: Define and Trigger the Search - Input field for topic to monitor                                                                                                                                                                  |
| Google Search Results   | HTTP Request                        | Searches Google via Bright Data for Quora links | Define Search Keyword | Links from Google HTML     | Section 2: Scrape Quora Discussion Links via Google - Scrapes Google search results bypassing anti-bot using Bright Data                                                                                                                    |
| Links from Google HTML  | HTML Extractor                      | Extracts thread titles and URLs from HTML      | Google Search Results  | Format Thread URLs         | Section 2: Scrape Quora Discussion Links via Google - Extracts thread links and titles                                                                                                                                                        |
| Format Thread URLs      | Code                               | Formats extracted titles and URLs into objects | Links from Google HTML | Individual Quora Threads   | Section 2: Scrape Quora Discussion Links via Google - Cleans and structures thread URLs                                                                                                                                                       |
| Individual Quora Threads| HTTP Request                       | Fetches full HTML of each Quora thread          | Format Thread URLs     | Answers from Threads       | Section 3: Scrape Actual Quora Answers - Scrapes full thread HTML using Bright Data                                                                                                                                                          |
| Answers from Threads    | HTML Extractor                     | Extracts answers/comments from thread HTML      | Individual Quora Threads| AI Feedback Summary1      | Section 3: Scrape Actual Quora Answers - Extracts user feedback from thread pages                                                                                                                                                            |
| AI Feedback Summary1    | LangChain Agent                    | Summarizes extracted feedback using OpenAI     | Answers from Threads   | Save to Google Sheet       | Section 4: Summarize & Store the Insights - AI generates summary of consumer feedback                                                                                                                                                        |
| OpenAI Chat Model       | LangChain LLM (GPT-4o-mini)        | Underlying model for AI summarization           | AI Feedback Summary1 (internal) | AI Feedback Summary1 (internal) | Section 4: Summarize & Store the Insights - GPT-4o-mini model used                                                                                                                                                                           |
| Save to Google Sheet    | Google Sheets                      | Appends summarized data into Google Sheets      | AI Feedback Summary1   | None                      | Section 4: Summarize & Store the Insights - Saves summary and topic for reporting                                                                                                                                                            |
| Sticky Note             | Sticky Note                       | Documentation and notes                          | None                  | None                      | Multiple detailed sticky notes provide section breakdowns, usage explanations, and links for support and resources                                                                                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 09:00 AM (adjust timezone if needed).

2. **Add a Set Node "Define Search Keyword"**  
   - Assign a string variable `keyword` with the value of the product/topic to monitor (e.g., `"iPhone+16"`).  
   - Connect Schedule Trigger output to this node.

3. **Add an HTTP Request Node "Google Search Results"**  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Authentication: Add header `Authorization: Bearer API_KEY` (replace `API_KEY` with your Bright Data token).  
   - Body Type: JSON  
   - JSON Body:  
     ```json
     {
       "zone": "n8n_unblocker",
       "url": "https://www.google.com/search?q=site:quora.com+{{ $json.keyword }}",
       "country": "us",
       "format": "raw",
       "headers": {
         "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
       }
     }
     ```  
   - Connect "Define Search Keyword" output to this node.

4. **Add an HTML Extract Node "Links from Google HTML"**  
   - Operation: Extract HTML Content  
   - Extraction Values:  
     - Key: `Thread Titles`, CSS Selector: `h3`, Return as Array: Yes  
     - Key: `b`, CSS Selector: `div.yuRUbf a`, Return as Array: Yes, Return Value: attribute (href)  
   - Connect "Google Search Results" output to this node.

5. **Add a Code Node "Format Thread URLs"**  
   - JavaScript code to pair titles and URLs:  
     ```js
     const titles = items[0].json["Thread Titles"];
     const urls = items[0].json["b"].map(obj => obj.href);

     const output = [];
     for (let i = 0; i < Math.min(titles.length, urls.length); i++) {
       output.push({
         json: {
           title: titles[i],
           url: urls[i]
         }
       });
     }
     return output;
     ```  
   - Connect "Links from Google HTML" output to this node.

6. **Add another HTTP Request Node "Individual Quora Threads"**  
   - Method: POST  
   - URL: `https://api.brightdata.com/request`  
   - Authentication: Header `Authorization: Bearer API_KEY` (same or another valid Bright Data key).  
   - JSON Body:  
     ```json
     {
       "zone": "n8n_unblocker",
       "url": "{{ $json.url }}",
       "country": "us",
       "format": "raw",
       "headers": {
         "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64)"
       }
     }
     ```  
   - Connect "Format Thread URLs" output to this node.

7. **Add an HTML Extract Node "Answers from Threads"**  
   - Operation: Extract HTML Content  
   - Extraction Values:  
     - Key: `answers`  
     - CSS Selector: Use the selector matching answer content on Quora (example given in original was a styled span; adjust as necessary to match Quora's current layout)  
     - Return as Array: Yes  
   - Connect "Individual Quora Threads" output to this node.

8. **Add a LangChain Agent Node "AI Feedback Summary1"**  
   - Text: `Summarize the following consumer feedback from Quora:\n{{ $json.answers }}`  
   - Credentials: Configure with OpenAI API credentials.  
   - Connect "Answers from Threads" output to this node.

9. **Add a LangChain LLM Node "OpenAI Chat Model"**  
   - Model: Select or input "gpt-4o-mini" or preferred OpenAI model.  
   - Credentials: Same OpenAI credentials as above.  
   - Connect internally to "AI Feedback Summary1" agent node (LangChain manages this).

10. **Add a Google Sheets Node "Save to Google Sheet"**  
    - Operation: Append  
    - Document ID: Provide your Google Sheet ID.  
    - Sheet Name: Use the first sheet or specify by gid (e.g., `gid=0`).  
    - Columns Mapping:  
      - `Topic`: Use expression `{{ $('Define Search Keyword').item.json.keyword }}`  
      - `Discussion Summary`: Use expression `{{ $json.output }}` (from AI summary node’s output)  
    - Credentials: Configure Google Sheets OAuth2 credentials with edit permissions.  
    - Connect "AI Feedback Summary1" output to this node.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                           |
|---------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Workflow automates forum monitoring powered by Bright Data, n8n, OpenAI, and Google Sheets.                    | General workflow description                                                                              |
| Workflow bypasses anti-bot restrictions on Google and Quora using Bright Data’s proxy service.                 | Bright Data integration                                                                                   |
| AI summarization uses OpenAI GPT-4o-mini language model via LangChain agent for human-like feedback summaries. | OpenAI API integration                                                                                     |
| Results are saved for easy access and sharing via Google Sheets.                                               | Google Sheets integration                                                                                  |
| For support or questions, contact: Yaron@nofluff.online                                                       | Support contact                                                                                           |
| Additional tips and video tutorials available at https://www.youtube.com/@YaronBeen/videos and LinkedIn profile https://www.linkedin.com/in/yaronbeen/ | Support and learning resources                                                                              |
| Bright Data affiliate link included for support: https://get.brightdata.com/1tndi4600b25                        | Affiliate program                                                                                          |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data processed is legal and publicly accessible.