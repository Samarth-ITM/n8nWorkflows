Find, Scrape & Analyze Twitter Posts by Name with Bright Data & Gemini

https://n8nworkflows.xyz/workflows/find--scrape---analyze-twitter-posts-by-name-with-bright-data---gemini-4325


# Find, Scrape & Analyze Twitter Posts by Name with Bright Data & Gemini

### 1. Workflow Overview

This workflow automates the process of finding, scraping, and analyzing Twitter (now called X) posts related to a specific person based on their full name. The workflow starts with user input via a web form, performs a Google search to locate the Twitter profile URL, uses Bright Data's scraping service to extract posts within a specified date range, analyzes the post content with Google's Gemini AI model, and finally saves the summarized insights alongside raw post data into a Google Sheet.

**Target Use Cases:**  
- Marketing and social media analysts wanting to monitor public Twitter profiles.  
- Outreach teams needing summarized insights from social media posts.  
- Data analysts automating social media data collection and sentiment analysis.

**Logical Blocks:**

- **1.1 Input Reception:** Receives user input with person’s full name and date range.  
- **1.2 Google Search URL Construction:** Builds a Google search URL to find the Twitter profile.  
- **1.3 Twitter Profile Extraction:** Uses Bright Data and AI to parse the Twitter profile URL from search results.  
- **1.4 Profile Existence Check:** Determines if the Twitter profile was found or not and branches accordingly.  
- **1.5 Bright Data Snapshot Request & Polling:** Requests post data snapshot from Bright Data, polls until data is ready.  
- **1.6 Data Extraction and Error Handling:** Extracts post data, handles errors, and prepares posts for analysis.  
- **1.7 AI Analysis of Posts:** Uses Google Gemini to analyze posts for tone, sentiment, topics, and popularity.  
- **1.8 Data Storage:** Appends extracted posts and AI summary into a Google Sheet.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Input Reception

- **Overview:** Captures user input via a web form including the person's full name and date range for analyzing posts.
- **Nodes Involved:**  
  - When User Completes Form

- **Node Details:**

  - **When User Completes Form**  
    - *Type & Role:* Form Trigger — entry point that triggers workflow on form submission.  
    - *Config:*  
      - Path: `search-user`  
      - Button: "Get References"  
      - Form fields: "Person Fullname" (required), "Analyze Posts From Date", "Analyze Posts To Date"  
      - Response Mode: last node output  
      - Form Title & Description provided for user guidance.  
    - *Input/Output:* No input; outputs form data JSON with user inputs.  
    - *Edge Cases:* User submits incomplete or invalid date format; handled by required field on fullname only.  
    - *Version:* 2.2

---

#### 2.2 Google Search URL Construction

- **Overview:** Constructs a Google search URL to find the Twitter profile of the given person by name.
- **Nodes Involved:**  
  - Edit Url Google Search

- **Node Details:**

  - **Edit Url Google Search**  
    - *Type & Role:* Set node — formats and sets a Google search URL string.  
    - *Config:* Creates a search query string of the form `https://www.google.com/search?q=site%3Ax.com+{encoded full name}`  
    - *Expressions:* Uses `encodeURIComponent` on the trimmed "Person Fullname" value from form input.  
    - *Input:* From Form Trigger node.  
    - *Output:* JSON property `google_search` with the constructed URL.  
    - *Edge Cases:* URL encoding failure unlikely; empty fullname blocked by form field requirement.  
    - *Version:* 3.4

---

#### 2.3 Twitter Profile Extraction

- **Overview:** Uses Bright Data to fetch search results, extract HTML content, and leverages Google Gemini AI to parse the Twitter profile URL and metadata.
- **Nodes Involved:**  
  - BrightData  
  - Extract Body and Title from Website  
  - Google Gemini Chat Model  
  - Structured Output Parser  
  - Parse X Url

- **Node Details:**

  - **BrightData**  
    - *Type & Role:* Bright Data scraping node — performs a web scrape of Google search results page.  
    - *Config:* Uses `web_unlocker1` zone; requests JSON format; country set to US; URL is from `google_search` property.  
    - *Input:* From Edit Url Google Search node.  
    - *Output:* Scraped Google search results page content.  
    - *Edge Cases:* Possible scraping failures, IP blocking, or quota limits from Bright Data.  
    - *Version:* 1  

  - **Extract Body and Title from Website**  
    - *Type & Role:* HTML extraction node — extracts the `<title>` and `<body>` HTML elements from scraped page.  
    - *Config:* CSS selectors `title` and `body`; trims extracted values.  
    - *Input:* From BrightData node.  
    - *Output:* JSON with extracted `title` and `body` strings.  
    - *Edge Cases:* Unexpected HTML structure or empty content if scrape failed.  
    - *Version:* 1.2  

  - **Google Gemini Chat Model**  
    - *Type & Role:* AI language model node — sends extracted HTML body content to Gemini for analysis.  
    - *Config:* Model used: `models/gemini-2.0-flash`.  
    - *Input:* From Extract Body and Title node (under ai_languageModel connection).  
    - *Output:* Raw AI chat output.  
    - *Edge Cases:* API rate limits, network issues, or malformed input causing AI failure.  
    - *Version:* 1  

  - **Structured Output Parser**  
    - *Type & Role:* Parses AI text output into structured JSON.  
    - *Config:* Expects JSON schema with `username`, `name`, `url`, `description` fields.  
    - *Input:* From Google Gemini Chat Model.  
    - *Output:* Parsed Twitter profile data as JSON array.  
    - *Edge Cases:* AI output not matching schema, parse failures.  
    - *Version:* 1.2  

  - **Parse X Url**  
    - *Type & Role:* Chain LLM node — further processes extracted body text to validate and extract Twitter profile URL matching the person’s fullname.  
    - *Config:* Sends body text and prompt instructing to extract x.com profile with a "match" property if matched to fullname. Uses output parser.  
    - *Input:* From Extract Body and Title from Website node.  
    - *Output:* Parsed profile data JSON with match property.  
    - *Edge Cases:* No matching profile found; malformed or ambiguous search results.  
    - *Version:* 1.6  

---

#### 2.4 Profile Existence Check

- **Overview:** Determines whether a valid Twitter profile URL was found and branches workflow accordingly.
- **Nodes Involved:**  
  - X Profile is Found?  
  - Form Not Found

- **Node Details:**

  - **X Profile is Found?**  
    - *Type & Role:* If node — checks if profile URL exists in parsed output.  
    - *Config:* Condition checks if `output[0]['url']` exists (not empty).  
    - *Input:* From Parse X Url node.  
    - *Output:* True branch continues workflow; False branch triggers "Form Not Found".  
    - *Edge Cases:* Empty or invalid URL; false negatives if AI parsing failed.  
    - *Version:* 2.2  

  - **Form Not Found**  
    - *Type & Role:* Form node — responds to user with message that no profile was found.  
    - *Config:* Responds with text: `We didn't found X Profile for "{Person Fullname}"`.  
    - *Input:* From X Profile is Found? node (false branch).  
    - *Output:* Ends interaction for negative cases.  
    - *Version:* 1  

---

#### 2.5 Bright Data Snapshot Request & Polling

- **Overview:** Requests Bright Data snapshot for Twitter posts filtered by profile URL and date range; polls repeatedly until data is ready.
- **Nodes Involved:**  
  - Snapshot Request  
  - Wait 30s - Polling Bright Data  
  - Snapshot Progress  
  - If - Checking status of Snapshot - if data is ready or not  
  - Wait 30s - Check if the snapshot is ready  
  - Snapshot Content

- **Node Details:**

  - **Snapshot Request**  
    - *Type & Role:* Bright Data marketplace dataset filter — requests snapshot of Twitter posts filtered by URL and date range from user.  
    - *Config:* Uses dataset "X (formerly Twitter) - Posts"; filters:  
      - `url` includes parsed Twitter profile URL  
      - `date_posted` >= "Analyze Posts From Date"  
      - `date_posted` <= "Analyze Posts To Date"  
    - *Output:* Snapshot ID for polling.  
    - *Edge Cases:* Invalid date range, empty URL, Bright Data API limits.  
    - *Version:* 1  

  - **Wait 30s - Polling Bright Data**  
    - *Type & Role:* Wait node — delays 30 seconds before next poll.  
    - *Config:* Wait 30 seconds.  
    - *Input:* From Snapshot Request.  
    - *Output:* Triggers Snapshot Progress.  
    - *Version:* 1.1  

  - **Snapshot Progress**  
    - *Type & Role:* Bright Data metadata — fetches snapshot metadata including status.  
    - *Input:* From Wait 30s - Polling Bright Data.  
    - *Output:* Status info.  
    - *Version:* 1  

  - **If - Checking status of Snapshot - if data is ready or not**  
    - *Type & Role:* If node — checks if snapshot status equals "ready".  
    - *Output:*  
      - True: proceeds to Snapshot Content node.  
      - False: waits 30s again (loops polling).  
    - *Version:* 2.2  

  - **Wait 30s - Check if the snapshot is ready**  
    - *Type & Role:* Wait node — additional wait before fetching content.  
    - *Input:* From "not ready" branch of snapshot status check.  
    - *Output:* Loops back to Snapshot Content.  
    - *Version:* 1.1  

  - **Snapshot Content**  
    - *Type & Role:* Bright Data get snapshot content — retrieves actual post data once snapshot is ready.  
    - *Input:* From ready status branch.  
    - *Output:* Raw post items.  
    - *Version:* 1  

---

#### 2.6 Data Extraction and Error Handling

- **Overview:** Processes snapshot content JSON, extracts relevant post fields, and manages error states.
- **Nodes Involved:**  
  - Check for errors1  
  - Check for errors  
  - Code  
  - Form Error

- **Node Details:**

  - **Check for errors1**  
    - *Type & Role:* If node — checks if `items` array exists in snapshot content (indicates data presence).  
    - *Output:*  
      - True: waits 30s and loops (snapshot still building).  
      - False: proceeds to Check for errors node.  
    - *Version:* 2.2  

  - **Check for errors**  
    - *Type & Role:* If node — verifies if JSON contains error property or empty response.  
    - *Output:*  
      - True: proceeds to Code node to extract posts.  
      - False: triggers Form Error.  
    - *Version:* 2.2  

  - **Code**  
    - *Type & Role:* Code node — extracts and normalizes posts data into structured JSON objects with post details like date, description, hashtags, likes, quotes, etc.  
    - *Config:* JavaScript code that maps over `items` array and extracts fields including nested quoted posts and tagged users.  
    - *Input:* From Check for errors node (true branch).  
    - *Output:* JSON with `posts` array.  
    - *Edge Cases:* Missing fields in posts handled by null defaults.  
    - *Version:* 2  

  - **Form Error**  
    - *Type & Role:* Form node — sends error message back to user if Bright Data reported error.  
    - *Config:* Responds with text: `Bright Data error for "{Person Fullname}"`.  
    - *Input:* From Check for errors node (false branch).  
    - *Output:* Ends workflow with error message.  
    - *Version:* 1  

---

#### 2.7 AI Analysis of Posts

- **Overview:** Analyzes extracted Twitter posts using Google Gemini AI to detect interests, tone, sentiments, topics, and popularity, producing a summary.
- **Nodes Involved:**  
  - Analyze Posts  
  - Google Gemini Chat Model1

- **Node Details:**

  - **Analyze Posts**  
    - *Type & Role:* Chain LLM node — sends posts JSON to the AI analysis prompt.  
    - *Config:* Prompt instructs AI to analyze posts for interests, tone, sentiments, common topics, popularity, and return a summary.  
    - *Input:* From Code node output.  
    - *Output:* AI-generated summary text.  
    - *Version:* 1.6  

  - **Google Gemini Chat Model1**  
    - *Type & Role:* AI language model node — runs Gemini model to process Analyze Posts node's prompt.  
    - *Config:* Model: `models/gemini-2.0-flash`.  
    - *Input:* From Analyze Posts node (ai_languageModel connection).  
    - *Output:* AI text summary.  
    - *Edge Cases:* API limits or failures.  
    - *Version:* 1  

---

#### 2.8 Data Storage

- **Overview:** Appends the person’s name, date range, Twitter URL, extracted posts, and AI summary into a Google Sheets spreadsheet.
- **Nodes Involved:**  
  - Google Sheets - Adding Posts and Summary

- **Node Details:**

  - **Google Sheets - Adding Posts and Summary**  
    - *Type & Role:* Google Sheets node — appends new row with all collected data.  
    - *Config:*  
      - Sheet name: `gid=0` (default first sheet)  
      - Document ID: provided Google Sheet ID  
      - Columns: `name`, `start_date`, `end_date`, `x_url`, `posts`, `ai_summary`  
      - Values pulled from various nodes including form input, parsed URL, extracted posts, and AI summary text.  
    - *Input:* From Analyze Posts node output (main connection).  
    - *Output:* Confirms append operation.  
    - *Edge Cases:* Google API authentication errors, sheet permission issues, data format mismatches.  
    - *Version:* 4.3  

---

### 3. Summary Table

| Node Name                          | Node Type                              | Functional Role                              | Input Node(s)                           | Output Node(s)                           | Sticky Note                                                    |
|-----------------------------------|--------------------------------------|----------------------------------------------|---------------------------------------|-----------------------------------------|---------------------------------------------------------------|
| When User Completes Form           | formTrigger                          | Receive user input with fullname & date range | -                                     | Edit Url Google Search                   |                                                               |
| Edit Url Google Search             | set                                  | Construct Google search URL for Twitter profile | When User Completes Form              | BrightData                              |                                                               |
| BrightData                        | brightData                          | Scrape Google search results page             | Edit Url Google Search                 | Extract Body and Title from Website     |                                                               |
| Extract Body and Title from Website| html                                | Extract title and body HTML from scraped page | BrightData                           | Parse X Url; Google Gemini Chat Model   |                                                               |
| Google Gemini Chat Model           | lmChatGoogleGemini                   | AI analysis to help extract Twitter profile   | Extract Body and Title from Website (ai_languageModel) | Parse X Url                             |                                                               |
| Structured Output Parser           | outputParserStructured               | Parse AI output into structured JSON           | Google Gemini Chat Model               | Parse X Url                             |                                                               |
| Parse X Url                       | chainLlm                            | Further extract and validate Twitter profile URL | Extract Body and Title from Website  | X Profile is Found?                      |                                                               |
| X Profile is Found?                | if                                  | Check if Twitter profile URL found              | Parse X Url                         | Snapshot Request; Form Not Found         |                                                               |
| Form Not Found                    | form                                | Respond to user that no profile was found       | X Profile is Found?                    | -                                       |                                                               |
| Snapshot Request                  | brightData                          | Request Bright Data snapshot for Twitter posts  | X Profile is Found?                    | Wait 30s - Polling Bright Data           |                                                               |
| Wait 30s - Polling Bright Data    | wait                                | Wait 30 seconds between polling snapshot status | Snapshot Request                    | Snapshot Progress                        |                                                               |
| Snapshot Progress                | brightData                          | Get snapshot metadata/status                     | Wait 30s - Polling Bright Data        | If - Checking status of Snapshot         |                                                               |
| If - Checking status of Snapshot  | if                                  | Check if snapshot is ready                        | Snapshot Progress                    | Snapshot Content; Wait 30s - Polling Bright Data |                                                                |
| Wait 30s - Check if the snapshot is ready | wait                           | Wait 30 seconds before next snapshot check       | Check for errors1 (not ready branch)  | Snapshot Content                        |                                                               |
| Snapshot Content                 | brightData                          | Retrieve snapshot content (Twitter posts)        | If - Checking status of Snapshot (ready branch); Wait 30s - Check if the snapshot is ready | Check for errors1 |                                                      |
| Check for errors1                | if                                  | Check if snapshot content has `items` array      | Snapshot Content                     | Wait 30s - Check if the snapshot is ready; Check for errors |                                                               |
| Check for errors                 | if                                  | Verify if data contains error property            | Check for errors1                   | Code; Form Error                        |                                                               |
| Code                            | code                                | Extract and normalize post data                    | Check for errors (true branch)       | Analyze Posts                          |                                                               |
| Form Error                     | form                                | Respond with Bright Data error message             | Check for errors (false branch)      | -                                     |                                                               |
| Analyze Posts                   | chainLlm                            | Analyze posts content with AI                        | Code                               | Google Gemini Chat Model1               |                                                               |
| Google Gemini Chat Model1        | lmChatGoogleGemini                   | Run Gemini AI model for posts analysis              | Analyze Posts (ai_languageModel)    | Google Sheets - Adding Posts and Summary |                                                               |
| Google Sheets - Adding Posts and Summary | googleSheets                   | Append analyzed data and posts to Google Sheets      | Analyze Posts                      | -                                     |                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node:**  
   - Type: formTrigger  
   - Configure webhook path: `search-user`  
   - Add form fields:  
     - Person Fullname (required)  
     - Analyze Posts From Date (optional)  
     - Analyze Posts To Date (optional)  
   - Button label: "Get References"  
   - Form title and description as given.

2. **Create Set node to build Google search URL:**  
   - Type: set  
   - Add assignment:  
     - Name: `google_search`  
     - Value: `https://www.google.com/search?q=site%3Ax.com+{{encodeURIComponent($json["Person Fullname"].trim())}}`

3. **Connect Form Trigger → Set node**

4. **Add Bright Data node to scrape Google results:**  
   - Resource: marketplaceDataset  
   - Operation: getSnapshotContent OR use Bright Data web unlocker zone  
   - URL: from `google_search` property  
   - Zone: `web_unlocker1`  
   - Format: JSON  
   - Country: US

5. **Connect Set node → Bright Data node**

6. **Add HTML extraction node:**  
   - Operation: extractHtmlContent  
   - Extraction values:  
     - key: `title`, CSS selector: `title`  
     - key: `body`, CSS selector: `body`  
   - Trim values enabled.

7. **Connect Bright Data → HTML extraction node**

8. **Add Google Gemini Chat Model node:**  
   - Model name: `models/gemini-2.0-flash`

9. **Connect HTML extraction node (ai_languageModel) → Google Gemini Chat Model**

10. **Add Structured Output Parser node:**  
    - JSON schema example:  
      ```json
      [{
        "username": "string",
        "name": "string",
        "url": "string",
        "description": "string"
      }]
      ```

11. **Connect Google Gemini Chat Model (ai_outputParser) → Structured Output Parser**

12. **Add Chain LLM node (Parse X Url):**  
    - Text input: extracted body from HTML node  
    - Prompt: extract x.com profile JSON matching fullname, add "match" property.  
    - Enable output parser.

13. **Connect HTML extraction node → Parse X Url node**

14. **Add If node (X Profile is Found?):**  
    - Condition: Check if `output[0].url` exists and is not empty.

15. **Connect Parse X Url → If node**

16. **Add Form node (Form Not Found):**  
    - Response text: `We didn't found X Profile for "{{ Person Fullname }}"`

17. **Connect If node (false branch) → Form Not Found**

18. **Add Bright Data Snapshot Request node:**  
    - Dataset ID: `gd_lwxkxvnf1cynvib9co` (X Twitter posts)  
    - Filter:  
      - URL includes parsed Twitter profile URL  
      - date_posted >= Analyze Posts From Date  
      - date_posted <= Analyze Posts To Date

19. **Connect If node (true branch) → Snapshot Request**

20. **Add Wait node (Wait 30s - Polling Bright Data):**  
    - Wait 30 seconds.

21. **Connect Snapshot Request → Wait node**

22. **Add Bright Data Snapshot Progress node:**  
    - Use snapshot_id from Snapshot Request output.

23. **Connect Wait node → Snapshot Progress**

24. **Add If node (Check if snapshot status is "ready"):**  
    - Condition: `$json.status == 'ready'`

25. **Connect Snapshot Progress → If node**

26. **Connect If node true branch → Snapshot Content:**  
    - Bright Data get snapshot content, snapshot_id from previous node.

27. **Connect If node false branch → Wait 30s - Polling Bright Data** (loop)

28. **Add Wait node (Wait 30s - Check if snapshot ready):**  
    - Wait 30 seconds.

29. **Connect Check for errors1 node false branch → Wait 30s - Check if snapshot ready**

30. **Connect Wait 30s - Check if snapshot ready → Snapshot Content**

31. **Add If node (Check for errors1):**  
    - Check if `items` array exists in snapshot content.

32. **Connect Snapshot Content → Check for errors1**

33. **Add If node (Check for errors):**  
    - Check if `$json.error` exists or empty.

34. **Connect Check for errors1 true branch → Check for errors**

35. **Add Code node:**  
    - JavaScript to extract posts and nested fields into structured JSON.

36. **Connect Check for errors true branch → Code**

37. **Add Form node (Form Error):**  
    - Respond with Bright Data error message.

38. **Connect Check for errors false branch → Form Error**

39. **Add Chain LLM node (Analyze Posts):**  
    - Input: posts JSON from Code node  
    - Prompt: Analyze twitter posts for interests, tone, sentiments, common topics, popularity, summary.

40. **Connect Code → Analyze Posts**

41. **Add Google Gemini Chat Model node:**  
    - Model: `models/gemini-2.0-flash`

42. **Connect Analyze Posts (ai_languageModel) → Google Gemini Chat Model**

43. **Add Google Sheets node:**  
    - Operation: Append  
    - Sheet name: `gid=0`  
    - Document ID: Your Google Sheet ID  
    - Columns: `name`, `start_date`, `end_date`, `x_url`, `posts`, `ai_summary`  
    - Map values from respective nodes (form input, parsed URL, extracted posts, AI summary).

44. **Connect Analyze Posts main output → Google Sheets node**

45. **Configure all credentials:**  
    - Bright Data API credentials.  
    - Google OAuth2 credentials for Google Sheets.  
    - Google Gemini API credentials for AI nodes.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                         |
|----------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| The workflow relies on Bright Data marketplace dataset ID `gd_lwxkxvnf1cynvib9co` for Twitter posts. | Bright Data dataset reference.                                         |
| Google Gemini model used is `models/gemini-2.0-flash`; ensure API access and quota are available.   | Gemini AI model specification.                                         |
| The Google Sheet document ID must be replaced with your own sheet ID that has appropriate permissions. | Google Sheets integration.                                             |
| Form trigger allows date inputs but does not enforce format beyond placeholders; validate externally. | Input validation recommendation.                                       |
| Bright Data scraping may be subject to rate limiting or IP blocking; handle retry and quota limits. | Potential integration limitations.                                     |
| The workflow is designed for English language analysis and Twitter (X) content.                     | Language and platform scope.                                           |
| The workflow is inactive by default (`active: false`). Activate to enable webhook triggers.          | n8n workflow activation note.                                         |

---

**Disclaimer:**  
This documentation is solely based on an automated workflow built with n8n integration and automation tool. All content processed respects applicable content policies and contains no illegal or protected data. All handled data is legal and publicly available.