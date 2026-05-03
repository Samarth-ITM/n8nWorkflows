Automated UPSC Current Affairs Digest from The Hindu to Google Sheets with Gemini AI

https://n8nworkflows.xyz/workflows/automated-upsc-current-affairs-digest-from-the-hindu-to-google-sheets-with-gemini-ai-9872


# Automated UPSC Current Affairs Digest from The Hindu to Google Sheets with Gemini AI

### 1. Workflow Overview

This workflow automates the daily compilation of UPSC-relevant current affairs from *The Hindu* newspaper and stores the curated data in a Google Sheet. It is designed to help UPSC aspirants, educators, and coaching institutes by filtering the most pertinent news, summarizing it, and categorizing it according to UPSC syllabus topics.

The workflow is logically divided into the following blocks:

- **1.1 Trigger and Initial Request:** Schedule or manual trigger initiates the workflow and retrieves the latest news homepage HTML from *The Hindu*.
- **1.2 Extraction of Article Links and Titles:** Parses the homepage HTML to extract URLs and titles of news articles.
- **1.3 Article Content Retrieval:** For each article URL, fetches the full article page and extracts the main article text.
- **1.4 AI Processing and Filtering:** Uses Google Gemini AI via a LangChain AI Agent to filter, analyze, summarize, and categorize articles based on UPSC relevance.
- **1.5 Data Storage:** Appends the structured summary and metadata of relevant articles into a designated Google Sheet.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Trigger and Initial Request

**Overview:**  
This block initiates the workflow either manually or automatically every day at 7 AM and fetches the latest news homepage HTML from *The Hindu*.

**Nodes Involved:**  
- Schedule Trigger: Daily at 7 AM  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- HTTP Request: Get The Hindu Front Page

**Node Details:**  

- **Schedule Trigger: Daily at 7 AM**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers the workflow once daily at 7 AM.  
  - *Configuration:* Set to trigger at hour 7 (7 AM) every day.  
  - *Inputs:* None  
  - *Outputs:* Triggers HTTP Request node  
  - *Failures:* None expected unless system time misconfiguration.  
  - *Note:* Sticky note clarifies it "Sets the time for the workflow to run every morning."

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual execution of the workflow on demand.  
  - *Configuration:* No parameters.  
  - *Inputs:* None  
  - *Outputs:* Triggers HTTP Request node  
  - *Failures:* None expected.

- **HTTP Request: Get The Hindu Front Page**  
  - *Type:* HTTP Request  
  - *Role:* Fetches the HTML content of *The Hindu* latest news homepage.  
  - *Configuration:*  
    - URL: https://www.thehindu.com/latest-news/  
    - Sends User-Agent header mimicking a modern browser to avoid blocking.  
  - *Inputs:* Trigger from Scheduler or Manual Trigger  
  - *Outputs:* Raw HTML content for parsing  
  - *Failures:* Network errors, HTTP errors, or website blocking.  
  - *Sticky Note:* "Sends a request to the latest news section of Newspaper Web."

---

#### 2.2 Extraction of Article Links and Titles

**Overview:**  
Parses the homepage HTML to extract all article URLs and titles for further processing.

**Nodes Involved:**  
- HTML: Extract Article Links & Titles  
- Code in JavaScript: Pair URL & Title

**Node Details:**  

- **HTML: Extract Article Links & Titles**  
  - *Type:* HTML Extract  
  - *Role:* Extracts URLs and titles from the news homepage HTML using CSS selectors.  
  - *Configuration:*  
    - Extract `href` attribute and text content from `h3.title > a` elements.  
    - Returns arrays of URLs and titles.  
  - *Inputs:* Raw HTML from HTTP Request node  
  - *Outputs:* JSON arrays of URLs and titles.  
  - *Failures:* Changes in website structure or CSS selectors may break extraction.  
  - *Sticky Note:* "Parses the received HTML."

- **Code in JavaScript: Pair URL & Title**  
  - *Type:* Code (JavaScript)  
  - *Role:* Pairs each URL with its corresponding title into individual items for iteration.  
  - *Configuration:* Custom JS code loops over extracted arrays and outputs paired objects.  
  - *Inputs:* Output from HTML Extraction node  
  - *Outputs:* Individual JSON items each with `url` and `title` fields.  
  - *Failures:* Mismatch in array lengths or missing data could cause pairing errors.  
  - *Sticky Note:* "Creates a separate data item for every single news article for individual processing."

---

#### 2.3 Article Content Retrieval

**Overview:**  
Fetches the full article page for each URL and extracts the clean article body text.

**Nodes Involved:**  
- HTTP Request1: Get Full Article Content  
- HTML1: Extract Article Body

**Node Details:**  

- **HTTP Request1: Get Full Article Content**  
  - *Type:* HTTP Request  
  - *Role:* Downloads the full HTML content of each news article URL.  
  - *Configuration:*  
    - URL is dynamically set from paired URL item (`={{ $json.url }}`)  
    - Sends User-Agent header to mimic browser.  
  - *Inputs:* From code node output (individual URL items)  
  - *Outputs:* Full HTML page of article  
  - *Failures:* Network issues, URL dead links, or access restrictions.  
  - *Sticky Note:* "Retrieve its full text/HTML content."

- **HTML1: Extract Article Body**  
  - *Type:* HTML Extract  
  - *Role:* Extracts the main article text from the full HTML page.  
  - *Configuration:*  
    - CSS selector targeting div with ID starting with `content-body-` to isolate main text.  
  - *Inputs:* Full article HTML from HTTP Request1  
  - *Outputs:* Extracted plain article text as `articleText`  
  - *Failures:* Website layout changes or selector mismatches may yield empty or incorrect extraction.  
  - *Sticky Note:* "Isolates and extracts the clean, primary article text."

---

#### 2.4 AI Processing and Filtering

**Overview:**  
Filters the articles to only those relevant for UPSC exams, summarizes them, extracts subject and importance, and prepares structured data.

**Nodes Involved:**  
- AI Agent: Filter & Analyze UPSC News  
- Google Gemini Chat Model (used internally by AI Agent)  
- Append row in sheet in Google Sheets1 (AI Tool connection)

**Node Details:**  

- **AI Agent: Filter & Analyze UPSC News**  
  - *Type:* LangChain AI Agent with Google Gemini Chat Model  
  - *Role:* Analyzes article content, filters to top 5-6 UPSC-relevant articles, summarizes, classifies subject, and provides importance note.  
  - *Configuration:*  
    - System message sets role as UPSC current affairs expert.  
    - Prompt requests structured JSON with article URL, brief summary, importance, and subject category.  
    - Rejects articles not in top 5-6 most important.  
  - *Inputs:* Article text extracted from previous node, plus paired URL.  
  - *Outputs:* Structured JSON with fields: Date, URL, Subject, Brief Summary, What is Important.  
  - *Failures:* API authentication errors, quota limits, prompt formatting issues, or incomplete AI responses.  
  - *Sub-workflow:* Uses Google Gemini Chat Model node internally.  
  - *Sticky Note:* "Filter for UPSC relevance, summarize, categorize, and prepare the data for the sheet."

- **Google Gemini Chat Model**  
  - *Type:* AI Language Model Node (Google Gemini)  
  - *Role:* Provides language model capabilities for the AI Agent.  
  - *Configuration:* Requires Google Gemini API credentials.  
  - *Inputs:* Prompt from AI Agent node.  
  - *Outputs:* AI-generated analysis and summary.  
  - *Failures:* API key issues, network errors, or model downtime.

- **Append row in sheet in Google Sheets1**  
  - *Type:* Google Sheets Tool node  
  - *Role:* Receives AI Agent data and appends it as a new row in the Google Sheet.  
  - *Configuration:*  
    - Maps AI output fields to sheet columns: Date, URL, Subject, Brief Summary, What is Important.  
    - Uses "append" operation.  
    - Requires Google Sheets credentials and correct sheet/document IDs.  
  - *Inputs:* Output from AI Agent node (ai_tool connection).  
  - *Outputs:* Confirmation of row append.  
  - *Failures:* Credential errors, sheet permission issues, or invalid sheet IDs.  
  - *Sticky Note:* None.

---

#### 2.5 Data Storage

**Overview:**  
Ensures that the structured summary data from AI is appended to the target Google Sheet for later reference.

**Nodes Involved:**  
- Append row in sheet

**Node Details:**  

- **Append row in sheet**  
  - *Type:* Google Sheets node  
  - *Role:* Inserts the filtered and summarized article data as new rows into the specified Google Sheet.  
  - *Configuration:*  
    - Defines columns explicitly: Date, URL, Subject, Brief Summary, What is Important.  
    - Appends data to the sheet identified by spreadsheet and sheet IDs (placeholders in config).  
  - *Inputs:* Output from AI Agent node main output (structured JSON).  
  - *Outputs:* Sheet update confirmation.  
  - *Failures:* Same as other Google Sheets node: permissions, invalid IDs, quota limits.  
  - *Sticky Note:* "Ensuring the structured data is physically written to the Google Sheet."

---

### 3. Summary Table

| Node Name                       | Node Type                         | Functional Role                                    | Input Node(s)                            | Output Node(s)                          | Sticky Note                                                                                      |
|--------------------------------|----------------------------------|---------------------------------------------------|-----------------------------------------|----------------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger: Daily at 7 AM | Schedule Trigger                 | Daily automated workflow start                     | None                                    | HTTP Request: Get The Hindu Front Page | Sets the time for the workflow to run every morning.                                           |
| When clicking ‘Execute workflow’| Manual Trigger                  | Manual workflow start                              | None                                    | HTTP Request: Get The Hindu Front Page |                                                                                                |
| HTTP Request: Get The Hindu Front Page | HTTP Request             | Fetch homepage HTML of *The Hindu* latest news   | Schedule Trigger, Manual Trigger        | HTML: Extract Article Links & Titles   | Sends a request to the latest news section of Newspaper Web.                                   |
| HTML: Extract Article Links & Titles | HTML Extract              | Extract article URLs and titles                    | HTTP Request: Get The Hindu Front Page  | Code in JavaScript: Pair URL & Title   | Parses the received HTML                                                                        |
| Code in JavaScript: Pair URL & Title | Code (JavaScript)           | Pair each URL with its title into individual items | HTML: Extract Article Links & Titles    | HTTP Request1: Get Full Article Content | Creates a separate data item for every single news article for individual processing             |
| HTTP Request1: Get Full Article Content | HTTP Request            | Fetch full article HTML content                    | Code in JavaScript: Pair URL & Title    | HTML1: Extract Article Body             | Retrieve its full text/HTML content                                                            |
| HTML1: Extract Article Body     | HTML Extract                    | Extract main article text                           | HTTP Request1: Get Full Article Content | AI Agent: Filter & Analyze UPSC News   | Isolates and extracts the clean, primary article text                                         |
| AI Agent: Filter & Analyze UPSC News | LangChain AI Agent          | Filter and analyze articles for UPSC relevance    | HTML1: Extract Article Body, Gemini Chat Model | Append row in sheet, Append row in sheet in Google Sheets1 | Filter for UPSC relevance, summarize, categorize, and prepare the data for the sheet           |
| Google Gemini Chat Model        | AI Language Model (Google Gemini) | Provides AI model for language understanding      | AI Agent: Filter & Analyze UPSC News    | AI Agent: Filter & Analyze UPSC News   |                                                                                                |
| Append row in sheet             | Google Sheets                   | Append structured article data to Google Sheet    | AI Agent: Filter & Analyze UPSC News    | None                                   | Ensuring the structured data is physically written to the Google Sheet                         |
| Append row in sheet in Google Sheets1 | Google Sheets Tool          | Alternative append action for AI output            | AI Agent: Filter & Analyze UPSC News    | None                                   |                                                                                                |
| Sticky Note                    | Sticky Note                    | Documentation and instructions                      | None                                    | None                                   | Automated Daily UPSC Current Affairs Compilation... (full content in node)                     |
| Sticky Note1                   | Sticky Note                    | Documentation of schedule trigger                   | None                                    | None                                   | Sets the time for the workflow to run every morning.                                           |
| Sticky Note2                   | Sticky Note                    | Documentation of HTTP Request                        | None                                    | None                                   | Sends a request to the latest news section of Newspaper Web.                                   |
| Sticky Note3                   | Sticky Note                    | Documentation of HTML parsing                        | None                                    | None                                   | Parses the received HTML                                                                        |
| Sticky Note4                   | Sticky Note                    | Documentation of code pairing URLs and titles      | None                                    | None                                   | Creates a separate data item for every single news article for individual processing             |
| Sticky Note5                   | Sticky Note                    | Documentation of article content fetch              | None                                    | None                                   | Retrieve its full text/HTML content                                                            |
| Sticky Note6                   | Sticky Note                    | Documentation of article extraction                  | None                                    | None                                   | Isolates and extracts the clean, primary article text                                         |
| Sticky Note7                   | Sticky Note                    | Documentation of AI filtering and analysis          | None                                    | None                                   | Filter for UPSC relevance, summarize, categorize, and prepare the data for the sheet           |
| Sticky Note8                   | Sticky Note                    | Documentation of Google Sheets append                | None                                    | None                                   | Ensuring the structured data is physically written to the Google Sheet                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**  
   - Add a **Schedule Trigger** node: Configure to run daily at 7 AM.  
   - Add a **Manual Trigger** node: No configuration needed.

2. **Create HTTP Request to Fetch Homepage:**  
   - Add an **HTTP Request** node named "Get The Hindu Front Page".  
   - Set URL to `https://www.thehindu.com/latest-news/`.  
   - Add a header for `User-Agent` with value resembling a modern browser (e.g., Chrome).  
   - Connect both triggers to this node.

3. **Parse Article Links and Titles:**  
   - Add an **HTML Extract** node named "Extract Article Links & Titles".  
   - Connect from HTTP Request node.  
   - Configure extraction:  
     - CSS selector: `h3.title > a`  
     - Extract attribute `href` as `url` (array).  
     - Extract text content as `title` (array).

4. **Pair URLs and Titles:**  
   - Add a **Code** node named "Pair URL & Title".  
   - Use JS code to loop through arrays and output paired JSON objects with `url` and `title`.  
   - Connect from HTML Extract node.

5. **Fetch Full Article Content:**  
   - Add an **HTTP Request** node named "Get Full Article Content".  
   - Set URL dynamically to `{{$json.url}}`.  
   - Copy User-Agent header from earlier request.  
   - Connect from Code node.

6. **Extract Article Body Text:**  
   - Add an **HTML Extract** node named "Extract Article Body".  
   - Connect from HTTP Request "Get Full Article Content".  
   - Configure CSS selector: `div[id^="content-body-"]` to extract article text as `articleText`.

7. **Set up Google Gemini Chat Model node:**  
   - Add **Google Gemini Chat Model** node.  
   - Configure with Google Gemini API credentials.

8. **Create AI Agent node:**  
   - Add **LangChain AI Agent** node named "Filter & Analyze UPSC News".  
   - Set system message: You are an expert in UPSC current affairs.  
   - Use prompt to analyze article content (`articleText`) and URL, filter top 5-6 most important articles, and produce JSON with Date, URL, Subject, Brief Summary, and What is Important.  
   - Configure to use Google Gemini Chat Model node as language model.  
   - Connect from HTML Extract "Extract Article Body" node.  
   - Connect AI Agent’s languageModel input to Google Gemini Chat Model node.

9. **Append Data to Google Sheet:**  
   - Add **Google Sheets** node named "Append row in sheet".  
   - Configure credentials for Google Sheets.  
   - Specify Spreadsheet ID and Sheet GID (replace placeholders with actual IDs).  
   - Define columns: Date, URL, Subject, Brief Summary, What is Important.  
   - Connect main output of AI Agent node to this node.

10. *(Optional)* Add **Google Sheets Tool** node named "Append row in sheet in Google Sheets1" for alternative append method and connect from AI Agent’s ai_tool output.

11. **Test the workflow:**  
   - Run manual trigger or wait for scheduled execution.  
   - Verify Google Sheet updates with filtered, analyzed news articles.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                         | Context or Link                                                                                           |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Automated Daily UPSC Current Affairs Compilation: This workflow scrapes *The Hindu*, uses Google Gemini AI to filter and summarize UPSC-relevant news, and stores the data in Google Sheets for easy daily review.                                                  | Full sticky note attached to the workflow, providing context and usage.                                  |
| Requires Google Gemini API Key and Google Sheets access credentials. Ensure your Google Sheet contains columns: Date, URL, Subject, Brief Summary, What is Important.                                                                                                | Setup instructions from sticky notes.                                                                   |
| Designed for UPSC aspirants, coaching institutes, and educators focusing on Governance, Economy, International Relations, Science & Technology, and Environment topics as per UPSC syllabus.                                                                        | Intended user audience.                                                                                   |
| For details on Google Gemini API usage and LangChain integration, refer to official Google and n8n documentation.                                                                                                                                                   | External reference, not included in workflow but recommended for advanced troubleshooting or modifications. |

---

This concludes the comprehensive reference documentation for the "Automated UPSC Current Affairs Digest from The Hindu to Google Sheets with Gemini AI" workflow. It covers the entire logic, node-by-node detail, reproduction instructions, and contextual notes to facilitate understanding and maintenance by advanced users and automation systems.