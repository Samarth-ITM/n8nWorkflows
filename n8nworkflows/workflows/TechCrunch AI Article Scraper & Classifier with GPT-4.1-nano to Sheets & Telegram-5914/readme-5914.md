TechCrunch AI Article Scraper & Classifier with GPT-4.1-nano to Sheets & Telegram

https://n8nworkflows.xyz/workflows/techcrunch-ai-article-scraper---classifier-with-gpt-4-1-nano-to-sheets---telegram-5914


# TechCrunch AI Article Scraper & Classifier with GPT-4.1-nano to Sheets & Telegram

### 1. Workflow Overview

This workflow automates the process of scraping the latest artificial intelligence news articles from TechCrunch, analyzing and classifying these articles using an AI language model (GPT-4.1-nano), storing the results in a Google Sheet, and sending summarized news updates via Telegram. It is designed to run daily, fetch news published the previous day, evaluate their content quality and categorization rigorously, and deliver concise, user-friendly summaries in Thai.

The workflow logic is organized into the following main blocks:

- **1.1 Schedule & Fetch News**: Trigger execution daily and retrieve AI news articles from TechCrunch.
- **1.2 Extract & Prepare URLs and Dates**: Parse article URLs and publish dates, and filter for articles published yesterday.
- **1.3 Article Content Retrieval & Wait**: Fetch full article content and introduce randomized delays to avoid rate-limiting.
- **1.4 AI Analysis & Classification**: Use GPT-4.1-nano to summarize, score, and categorize each article.
- **1.5 Data Storage & Notification**: Save structured results to Google Sheets and send a Telegram message with summarized news.

---

### 2. Block-by-Block Analysis

#### 2.1 Schedule & Fetch News

**Overview:**  
This block triggers the workflow daily at 6 AM Bangkok time and fetches the HTML content of the TechCrunch AI category page.

**Nodes Involved:**  
- Schedule Trigger1  
- TechCrunchNews-AI1  

**Node Details:**  
- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Configured to trigger once daily at 06:00 (Bangkok timezone)  
  - No inputs; outputs trigger signal to start the workflow  
  - Edge cases: If the n8n instance is down at the scheduled time, missed triggers may occur.

- **TechCrunchNews-AI1**  
  - Type: HTTP Request  
  - HTTP GET to `https://techcrunch.com/category/artificial-intelligence/` to retrieve the listing page of AI news  
  - No authentication required  
  - Input: Trigger from Schedule Trigger1  
  - Output: Raw HTML of the news listing page  
  - Potential issues: HTTP request failure, site downtime, or structure changes.

---

#### 2.2 Extract & Prepare URLs and Dates

**Overview:**  
Extracts article publication dates and URLs from the HTML, splits these into individual items, formats dates, and filters articles to only process those published yesterday.

**Nodes Involved:**  
- GetDate1  
- Split URL3  
- Edit Fields6  
- SelectDate  

**Node Details:**  
- **GetDate1**  
  - Type: HTML Extract  
  - Extracts two attributes from the HTML:  
    - `DatePublished` from the `datetime` attribute of `.loop-card__meta > time` elements  
    - `url` from `data-destinationlink` attribute of `.loop-card__title > a` elements  
  - Returns arrays of dates and URLs  
  - Input: HTML from TechCrunchNews-AI1  
  - Output: JSON containing arrays of `DatePublished` and `url`  
  - Edge cases: CSS selector changes on the source page, missing attributes, empty lists.

- **Split URL3**  
  - Type: Split Out  
  - Splits the arrays of `DatePublished` and `url` into individual JSON items for batch processing  
  - Input: JSON with arrays from GetDate1  
  - Output: Individual items each containing a single `DatePublished` and `url` pair

- **Edit Fields6**  
  - Type: Set  
  - Converts `DatePublished` string into a date object and formats it as `dd/MM/yyyy` for easier comparison  
  - Passes along the URL unchanged  
  - Input: Individual items from Split URL3  
  - Output: JSON with string-formatted `DatePublished` and `url`

- **SelectDate**  
  - Type: Switch  
  - Filters for articles where `DatePublished` equals yesterday’s date (based on workflow execution date)  
  - Routes “YesterdayOnly” articles for further processing, discards others  
  - Uses expression to calculate yesterday’s date dynamically  
  - Input: Formatted date and URL from Edit Fields6  
  - Output: Only articles published yesterday proceed downstream  
  - Edge cases: Date format mismatch, timezone differences, no articles matching yesterday.

---

#### 2.3 Article Content Retrieval & Wait

**Overview:**  
For each article published yesterday, fetch full article content, extract the main body, and insert a randomized wait time between requests to avoid overloading the source or hitting rate limits.

**Nodes Involved:**  
- Loop Over Items1  
- URL_Content  
- GetBody  
- TimeWaitRandom1  
- Wait1  

**Node Details:**  
- **Loop Over Items1**  
  - Type: Split In Batches  
  - Processes articles one by one (batch size = 1) to control flow  
  - Input: Articles filtered by SelectDate  
  - Output: Single article data item forwarded downstream

- **URL_Content**  
  - Type: HTTP Request  
  - Fetches the full HTML content of the article’s URL (from SelectDate)  
  - Input: Single article URL from Loop Over Items1  
  - Output: HTML content of the article page  
  - Edge cases: HTTP failures, redirects, content blocking

- **GetBody**  
  - Type: HTML Extract  
  - Extracts the main article content using CSS selector `div[class*="entry-content"]`  
  - Returns array of HTML content blocks (usually one) as `body`  
  - Input: HTML from URL_Content  
  - Output: Extracted article body content  
  - Edge cases: HTML structure changes, empty content

- **TimeWaitRandom1**  
  - Type: Code (JavaScript)  
  - Generates a random delay between 4 and 10 seconds to introduce throttling  
  - Outputs a JSON with `delay` value in seconds  
  - Input: Trigger from Loop Over Items1 (parallel to SetData)  
  - Output: Delay time passed to Wait1

- **Wait1**  
  - Type: Wait  
  - Pauses execution for the delay generated by TimeWaitRandom1 (to space out HTTP requests)  
  - Input: Delay seconds from TimeWaitRandom1  
  - Output: Trigger to continue after wait

---

#### 2.4 AI Analysis & Classification

**Overview:**  
Processes the extracted article content with an AI agent to generate a summary, score, and category in Thai language, parses the structured JSON output, then prepares the data for storage and notification.

**Nodes Involved:**  
- AI Agent1  
- Structured Output Parser1  
- SetData  
- Gpt-4.1-nano (subnode of AI Agent1)  

**Node Details:**  
- **AI Agent1**  
  - Type: LangChain AI Agent Node  
  - Uses prompt in Thai instructing the AI to:  
    1. Summarize the article in 3–4 concise sentences  
    2. Score the article between 1–100 based on importance, novelty, quality, and presentation, with penalties for ads or low-quality content  
    3. Categorize the article into one of 9 predefined categories (e.g., politics, technology)  
  - Input: Article body extracted by GetBody  
  - Output: AI-generated JSON containing headline, summary, score, and category  
  - Uses GPT-4.1-nano as the language model (see Gpt-4.1-nano node)  
  - Retries on fail enabled  
  - Edge cases: API rate limits, malformed AI responses, prompt misinterpretation

- **Gpt-4.1-nano**  
  - Type: LangChain OpenAI Chat LLM  
  - Model set to `gpt-4.1-nano`  
  - Connected internally as the AI Agent’s language model  
  - Credentials: OpenAI API key configured  
  - Potential issues: API key expiry, model availability

- **Structured Output Parser1**  
  - Type: LangChain Structured Output Parser  
  - Validates and parses AI output strictly according to the schema:  
    ```json
    {
      "headline": "หัวข้อ",
      "summary": "ข้อความสรุป...",
      "score": "75",
      "category": "การเมือง"
    }
    ```  
  - Input: Raw AI Agent output  
  - Output: JSON with parsed fields for downstream use  
  - Edge cases: Parsing errors if AI deviates from schema

- **SetData**  
  - Type: Set  
  - Maps parsed AI output fields (`headline`, `summary`, `score`, `category`) plus original article `url` and `DatePublished` for storage and notification  
  - Input: Parsed AI output and article metadata  
  - Output: Structured JSON ready for saving and messaging

---

#### 2.5 Data Storage & Notification

**Overview:**  
Appends the structured news data to a Google Sheet and sends a formatted summary message via Telegram chat.

**Nodes Involved:**  
- NewsData  
- Send a text message  

**Node Details:**  
- **NewsData**  
  - Type: Google Sheets Append Row  
  - Appends one row per article with columns: `url`, `score`, `summary`, `category`, `headline`, `DatePublished`  
  - Uses Google Sheets OAuth2 credentials  
  - Target Sheet ID: `1TcYe0JRUh_boWjpo7BLAB_9ylyEQV_GZK8CAEb73yWs`, sheet named “ชีต1” (gid=0)  
  - Input: Structured data from SetData  
  - Output: Confirmation of row append  
  - Edge cases: Credential expiry, API quota limits, sheet access permissions

- **Send a text message**  
  - Type: Telegram node  
  - Sends a message to chat ID `8163566489` with a Thai-language summary of the news item, including date, headline, summary, score, category, and URL  
  - Uses Telegram API credentials  
  - Input: Same structured data as NewsData  
  - Output: Telegram message sent confirmation  
  - Edge cases: Chat ID invalid, API limits, connectivity issues

---

### 3. Summary Table

| Node Name           | Node Type                             | Functional Role                       | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                      |
|---------------------|-------------------------------------|------------------------------------|------------------------|------------------------|-------------------------------------------------------------------------------------------------|
| Schedule Trigger1    | Schedule Trigger                    | Daily execution trigger             | –                      | TechCrunchNews-AI1     |                                                                                                 |
| TechCrunchNews-AI1   | HTTP Request                       | Fetch TechCrunch AI news page       | Schedule Trigger1      | GetDate1               |                                                                                                 |
| GetDate1            | HTML Extract                      | Extract article dates and URLs      | TechCrunchNews-AI1     | Split URL3             |                                                                                                 |
| Split URL3           | Split Out                        | Split arrays into individual items  | GetDate1               | Edit Fields6           |                                                                                                 |
| Edit Fields6         | Set                              | Format date string and pass URL     | Split URL3             | SelectDate             |                                                                                                 |
| SelectDate           | Switch                          | Filter articles published yesterday | Edit Fields6           | Loop Over Items1        |                                                                                                 |
| Loop Over Items1     | Split In Batches                 | Process articles one by one         | SelectDate             | SetData, TimeWaitRandom1 |                                                                                                 |
| SetData              | Set                              | Prepare AI output and metadata      | Loop Over Items1, AI Agent1 | NewsData               |                                                                                                 |
| TimeWaitRandom1      | Code                             | Generate random delay (4–10 sec)    | Loop Over Items1       | Wait1                  |                                                                                                 |
| Wait1                | Wait                             | Pause execution for delay            | TimeWaitRandom1        | URL_Content            |                                                                                                 |
| URL_Content          | HTTP Request                     | Fetch article full HTML             | Wait1                  | GetBody                |                                                                                                 |
| GetBody              | HTML Extract                    | Extract article main content         | URL_Content            | AI Agent1              |                                                                                                 |
| AI Agent1            | LangChain AI Agent              | Summarize, score, categorize article| GetBody, Structured Output Parser1, Gpt-4.1-nano | Loop Over Items1 |                                                                                                 |
| Structured Output Parser1 | LangChain Output Parser Structured | Parse AI JSON output strictly       | AI Agent1              | SetData                |                                                                                                 |
| Gpt-4.1-nano         | LangChain LLM Chat OpenAI       | GPT-4.1-nano language model          | AI Agent1 (ai_languageModel) | AI Agent1 (ai_languageModel) |                                                                                                 |
| NewsData              | Google Sheets                   | Append article data to Google Sheet  | SetData                 | Send a text message    |                                                                                                 |
| Send a text message  | Telegram                        | Send summary message via Telegram    | NewsData                | –                      |                                                                                                 |
| Sticky Note4          | Sticky Note                    | Informational note about US news timing | –                      | –                      | ## News coming from U.S. \nTimeDelayed 11 Hours Behind                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 06:00 (time zone: Asia/Bangkok)

2. **Add an HTTP Request node ("TechCrunchNews-AI1")**  
   - Method: GET  
   - URL: `https://techcrunch.com/category/artificial-intelligence/`  
   - Connect output of Schedule Trigger to this node

3. **Add an HTML Extract node ("GetDate1")**  
   - Operation: Extract HTML Content  
   - Extraction Values:  
     - Key: `DatePublished`, Attribute: `datetime`, CSS Selector: `.loop-card__meta > time`, Return Array: true, Return Value: attribute  
     - Key: `url`, Attribute: `data-destinationlink`, CSS Selector: `.loop-card__title > a`, Return Array: true, Return Value: attribute  
   - Connect TechCrunchNews-AI1 output to GetDate1

4. **Add a Split Out node ("Split URL3")**  
   - Field to Split Out: `DatePublished, url`  
   - Connect GetDate1 output to this node

5. **Add a Set node ("Edit Fields6")**  
   - Assignments:  
     - `DatePublished`: `={{ $json.DatePublished.toDateTime().format('dd/MM/yyyy') }}`  
     - `url`: `={{ $json.url }}`  
   - Connect Split URL3 output to Edit Fields6

6. **Add a Switch node ("SelectDate")**  
   - Rules:  
     - Output Key: `YesterdayOnly` where `DatePublished` equals yesterday’s date in `dd/MM/yyyy` format  
   - Connect Edit Fields6 output to SelectDate

7. **Add a Split In Batches node ("Loop Over Items1")**  
   - No special batch size setting needed (default 1)  
   - Connect SelectDate output (YesterdayOnly branch) to Loop Over Items1

8. **Add a Code node ("TimeWaitRandom1")**  
   - Code:  
     ```javascript
     const minDelay = 4; // seconds  
     const maxDelay = 10; // seconds  
     const randomDelay = Math.floor(Math.random() * (maxDelay - minDelay + 1)) + minDelay;  
     return [{ json: { delay: randomDelay } }];
     ```  
   - Connect Loop Over Items1 output (main) to TimeWaitRandom1

9. **Add a Wait node ("Wait1")**  
   - Wait Time: `={{ $json.delay.format() }}` (seconds)  
   - Connect TimeWaitRandom1 output to Wait1

10. **Add an HTTP Request node ("URL_Content")**  
    - Method: GET  
    - URL: `={{ $('SelectDate').item.json.url }}`  
    - Connect Wait1 output to URL_Content

11. **Add an HTML Extract node ("GetBody")**  
    - Operation: Extract HTML Content  
    - Extraction Value:  
      - Key: `body`, CSS Selector: `div[class*="entry-content"]`, Return Array: true  
    - Connect URL_Content output to GetBody

12. **Add a LangChain AI Agent node ("AI Agent1")**  
    - Prompt: Use the detailed Thai-language prompt for summarizing, scoring, and categorizing the article content as given in the original prompt  
    - Output parser: Enable structured output parser with schema matching the JSON keys: `headline`, `summary`, `score`, `category`  
    - Connect GetBody output to AI Agent1  
    - Configure retry on fail enabled

13. **Add a LangChain Structured Output Parser node ("Structured Output Parser1")**  
    - Set schema type to manual with the JSON schema as above  
    - Connect AI Agent1 output parser to this node

14. **Add a Set node ("SetData")**  
    - Assignments:  
      - `score`: `={{ $json.output.score }}`  
      - `headline`: `={{ $json.output.headline }}`  
      - `summary`: `={{ $json.output.summary }}`  
      - `category`: `={{ $json.output.category }}`  
      - `url`: `={{ $('SelectDate').item.json.url }}`  
      - `DatePublished`: `={{ $('SelectDate').item.json.DatePublished }}`  
    - Connect Structured Output Parser1 output to SetData

15. **Add a Google Sheets node ("NewsData")**  
    - Operation: Append  
    - Document ID: `1TcYe0JRUh_boWjpo7BLAB_9ylyEQV_GZK8CAEb73yWs`  
    - Sheet Name: `ชีต1` (gid=0)  
    - Map columns as per SetData fields: url, score, summary, category, headline, DatePublished  
    - Connect SetData output to NewsData  
    - Credentials: Configure Google Sheets OAuth2 credentials

16. **Add a Telegram node ("Send a text message")**  
    - Chat ID: `8163566489`  
    - Text:  
      ```
      สรุปข่าวใหม่ 📰 วันที่: {{$json.DatePublished}}

      หัวข้อ📣 : {{$json.headline}}

      สรุป💬 : {{$json.summary}}

      คะแนน⭐ : {{$json.score}}
      หมวดหมู่📁 : {{$json.category}}
      ลิงก์🔗 : {{$json.url}}
      ```  
    - Connect NewsData output to this node  
    - Credentials: Configure Telegram API credentials

17. **Connect Loop Over Items1 output to both SetData and TimeWaitRandom1**  
    - This sets parallel branches: one for processing AI data, one for managing delays

18. **Connect AI Agent1 output to Loop Over Items1**  
    - To enable continuous processing of next articles after AI analysis

---

### 5. General Notes & Resources

| Note Content                                    | Context or Link                                                  |
|------------------------------------------------|-----------------------------------------------------------------|
| News are from U.S. with a time delay of 11 hrs | Sticky Note in workflow near Edit Fields6 node                  |
| Source website: TechCrunch AI category page     | https://techcrunch.com/category/artificial-intelligence/        |
| Language model used: GPT-4.1-nano               | OpenAI GPT-4.1-nano configured via n8n LangChain nodes          |
| Telegram chat ID and Google Sheets document ID are specific to user setup | Adjust credentials and IDs according to deployment environment   |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow implemented in n8n, respecting all current content policies. It contains no illegal, offensive, or protected elements and manipulates only legal and public data.