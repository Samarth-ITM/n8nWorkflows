Create Daily Israeli Economic Newsletter using RSS and GPT-4o

https://n8nworkflows.xyz/workflows/workflow-3564-1747220137551.png


# Create Daily Israeli Economic Newsletter using RSS and GPT-4o

### 1. Workflow Overview

This workflow automates the creation and delivery of a daily Israeli economic newsletter in Hebrew, leveraging RSS feeds and OpenAI’s GPT-4o model. It is designed for professionals such as economists, analysts, investors, and policymakers who require a concise, relevant, and actionable summary of the day’s most important economic news, with a focus on Israeli sources and content presented in a right-to-left (RTL) Hebrew format.

The workflow runs daily at 8:00 PM Israel time and consists of the following logical blocks:

- **1.1 Data Collection:** Fetches the latest news articles from two Hebrew news sources (Calcalist and Mako) via RSS feeds.
- **1.2 Data Processing:** Cleans, filters, deduplicates, and sorts the collected articles, preparing them for AI analysis.
- **1.3 AI Selection:** Uses GPT-4o to select the top 5 most relevant articles tailored for a senior executive audience.
- **1.4 Article Summarization & Text Extraction:** Fetches full article HTML pages, extracts relevant text snippets, and formats summaries.
- **1.5 Email Generation & Delivery:** Creates a styled, responsive HTML email in Hebrew with proper RTL layout and sends it via SMTP to the target inbox.

---

### 2. Block-by-Block Analysis

#### 2.1 Data Collection

**Overview:**  
This block triggers the workflow daily at 8:00 PM Israel time and retrieves news articles from two RSS feeds: Calcalist and Mako.

**Nodes Involved:**  
- Schedule Trigger  
- RSS - Mako  
- RSS - Calcalist

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow daily at 20:00 (8 PM) Israel time.  
  - Configuration: Set to trigger at hour 20, minute null (exactly 8 PM).  
  - Inputs: None (trigger node)  
  - Outputs: Triggers RSS nodes and date node simultaneously.  
  - Edge Cases: Timezone misconfiguration could cause incorrect trigger time.

- **RSS - Mako**  
  - Type: RSS Feed Read  
  - Role: Fetches latest articles from Mako’s RSS feed.  
  - Configuration: URL set to https://storage.googleapis.com/mako-sitemaps/rss-hp.xml  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Raw RSS items to "Edit Fields - Mako" node  
  - Edge Cases: RSS feed downtime, malformed XML, SSL issues (ignoreSSL false).

- **RSS - Calcalist**  
  - Type: RSS Feed Read  
  - Role: Fetches latest articles from Calcalist’s RSS feed.  
  - Configuration: URL set to https://www.calcalist.co.il/GeneralRSS/0,16335,L-8,00.xml  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Raw RSS items to "Edit Fields - Calcalist" node  
  - Edge Cases: Same as above (feed availability, SSL, malformed data).

---

#### 2.2 Data Processing

**Overview:**  
This block cleans and normalizes the RSS data, removes duplicates, filters invalid entries, and sorts articles by publication date to prepare a clean list for AI selection.

**Nodes Involved:**  
- Edit Fields - Mako  
- Edit Fields - Calcalist  
- Merged RSS  
- Remove NaN  
- Sort List  
- Clean List

**Node Details:**

- **Edit Fields - Mako**  
  - Type: Set  
  - Role: Cleans and formats Mako RSS fields.  
  - Configuration:  
    - Title: Removes tags like “[PACK]” and trims whitespace.  
    - Link: Adjusts torrent download URLs to a standard format.  
    - pubDate: Converts publication date to Unix timestamp (milliseconds).  
  - Inputs: RSS - Mako  
  - Outputs: To Merged RSS node  
  - Edge Cases: Missing or malformed pubDate could cause NaN.

- **Edit Fields - Calcalist**  
  - Type: Set  
  - Role: Cleans and formats Calcalist RSS fields.  
  - Configuration:  
    - Title: Removes “[PACK]” prefix and video resolution tags, appends size info extracted from content HTML.  
    - Link: Adjusts torrent download URLs similarly.  
    - pubDate: Converts to Unix timestamp.  
  - Inputs: RSS - Calcalist  
  - Outputs: To Merged RSS node  
  - Edge Cases: Missing size info extraction could cause errors.

- **Merged RSS**  
  - Type: Merge  
  - Role: Combines Mako and Calcalist article lists into one stream.  
  - Inputs: From both Edit Fields nodes  
  - Outputs: To Remove NaN node  
  - Edge Cases: None significant.

- **Remove NaN**  
  - Type: Filter  
  - Role: Filters out articles with invalid or missing publication dates (NaN).  
  - Configuration: Filters out items where pubDate contains “NaN”.  
  - Inputs: Merged RSS  
  - Outputs: To Sort List  
  - Edge Cases: Articles missing pubDate or with malformed dates.

- **Sort List**  
  - Type: Sort  
  - Role: Sorts articles descending by publication date (newest first).  
  - Inputs: Remove NaN  
  - Outputs: To Clean List  
  - Edge Cases: None significant.

- **Clean List**  
  - Type: Code (JavaScript)  
  - Role:  
    - Removes duplicate articles by link.  
    - Limits to top 50 most recent articles.  
    - Formats a clean text list of titles and links for AI input.  
  - Key Logic: Uses Map to deduplicate, sorts by pubDate, slices top 50, formats numbered list with titles and links.  
  - Inputs: Sort List  
  - Outputs: To ChatGPT 4o node  
  - Edge Cases: Duplicate detection depends on exact link matching; malformed links may cause misses.

---

#### 2.3 AI Selection

**Overview:**  
This block sends the cleaned list of articles to OpenAI GPT-4o to select the top 5 most relevant articles for a senior executive, ensuring at least one article covers current affairs and security.

**Nodes Involved:**  
- ChatGPT 4o  
- Split Out

**Node Details:**

- **ChatGPT 4o**  
  - Type: OpenAI (LangChain)  
  - Role: Uses GPT-4o to analyze article titles and select top 5 relevant articles.  
  - Configuration:  
    - Model: gpt-4o  
    - System prompt instructs to select 5 articles relevant to a senior CEO, with at least one on current affairs/security.  
    - Input: Cleaned list of article titles and links as formatted text.  
    - Output: JSON array with article title and link.  
  - Inputs: Clean List  
  - Outputs: Split Out  
  - Credentials: OpenAI API key required.  
  - Edge Cases: API rate limits, malformed input, unexpected output format.

- **Split Out**  
  - Type: Split Out  
  - Role: Splits the JSON array of selected articles into individual items for further processing.  
  - Inputs: ChatGPT 4o  
  - Outputs: Fetch HTML  
  - Edge Cases: Empty or malformed JSON output from GPT.

---

#### 2.4 Article Summarization & Text Extraction

**Overview:**  
This block fetches the full HTML content of each selected article, extracts relevant text snippets (subtitles or headers), and formats a clean summary for the email.

**Nodes Involved:**  
- Fetch HTML  
- Extract Text  
- Clean Text  
- Aggregate

**Node Details:**

- **Fetch HTML**  
  - Type: HTTP Request  
  - Role: Downloads the full HTML page of each article using its URL.  
  - Inputs: Split Out  
  - Outputs: Extract Text  
  - Edge Cases: HTTP errors, timeouts, redirects, invalid URLs.

- **Extract Text**  
  - Type: HTML  
  - Role: Extracts specific text snippets from the HTML using CSS selectors.  
  - Configuration:  
    - Extracts two keys:  
      - `data-calcalist` from `.calcalistArticleHeader .subTitle`  
      - `data-mako` from `.article-header header h2`  
  - Inputs: Fetch HTML  
  - Outputs: Clean Text  
  - Edge Cases: Missing selectors, changed website structure, empty extraction.

- **Clean Text**  
  - Type: Set  
  - Role: Combines extracted text snippets into a summary, assigns title and URL for each article.  
  - Configuration:  
    - Sets `title` from the original article title.  
    - Sets `summary` by concatenating extracted text from Calcalist and Mako.  
    - Sets `url` from the article link.  
  - Inputs: Extract Text  
  - Outputs: Aggregate  
  - Edge Cases: Missing extracted text results in empty summaries.

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines all individual article summaries into a single JSON array for email generation.  
  - Inputs: Clean Text  
  - Outputs: Create HTML  
  - Edge Cases: Large payloads may affect performance.

---

#### 2.5 Email Generation & Delivery

**Overview:**  
This block generates a fully styled, responsive HTML email in Hebrew with proper RTL layout, embedding the selected articles and summaries, then sends it via SMTP.

**Nodes Involved:**  
- Create HTML  
- Get Date  
- Merge  
- Send Daily News

**Node Details:**

- **Create HTML**  
  - Type: HTML  
  - Role: Builds the complete HTML email body with embedded article titles, summaries, and links.  
  - Configuration:  
    - Uses inline CSS for styling and RTL direction.  
    - Dynamically inserts article data from aggregated JSON array.  
    - Includes a footer with branding text.  
  - Inputs: Aggregate  
  - Outputs: Merge  
  - Edge Cases: HTML rendering issues in some email clients.

- **Get Date**  
  - Type: Function  
  - Role: Generates the current date formatted for Israel timezone (Asia/Jerusalem) in DD/MM/YYYY format.  
  - Outputs: JSON with `date_today` field.  
  - Inputs: Schedule Trigger (parallel)  
  - Outputs: Merge  
  - Edge Cases: Timezone misconfiguration.

- **Merge**  
  - Type: Merge  
  - Role: Combines the HTML content and the current date into one JSON object for email sending.  
  - Inputs: Create HTML (main input), Get Date (second input)  
  - Outputs: Send Daily News  
  - Edge Cases: Synchronization issues if inputs arrive at different times.

- **Send Daily News**  
  - Type: Email Send  
  - Role: Sends the final HTML email to the configured recipient using SMTP.  
  - Configuration:  
    - Subject: Includes the current date in Hebrew format.  
    - To: Configured recipient email (e.g., Elay Guez <elay96@gmail.com>).  
    - From: Configured sender email (e.g., Elay's AI Assistant <elayguez@gmail.com>).  
    - Credentials: SMTP credentials (host, port, username, password).  
  - Inputs: Merge  
  - Edge Cases: SMTP authentication failures, network issues, email delivery failures.

---

### 3. Summary Table

| Node Name           | Node Type                 | Functional Role                                    | Input Node(s)                      | Output Node(s)           | Sticky Note                                                                                   |
|---------------------|---------------------------|---------------------------------------------------|----------------------------------|--------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger     | Schedule Trigger          | Triggers workflow daily at 8 PM Israel time       | None                             | RSS - Mako, RSS - Calcalist, Get Date |                                                                                              |
| RSS - Mako          | RSS Feed Read             | Fetches articles from Mako RSS feed                | Schedule Trigger                 | Edit Fields - Mako        | # Data Collection Fetches latest news articles from two RSS sources: Calcalist and Mako      |
| RSS - Calcalist     | RSS Feed Read             | Fetches articles from Calcalist RSS feed           | Schedule Trigger                 | Edit Fields - Calcalist   | # Data Collection Fetches latest news articles from two RSS sources: Calcalist and Mako      |
| Edit Fields - Mako  | Set                       | Cleans and formats Mako RSS fields                 | RSS - Mako                      | Merged RSS                |                                                                                              |
| Edit Fields - Calcalist | Set                    | Cleans and formats Calcalist RSS fields            | RSS - Calcalist                 | Merged RSS                |                                                                                              |
| Merged RSS          | Merge                     | Combines Mako and Calcalist article lists          | Edit Fields - Mako, Edit Fields - Calcalist | Remove NaN                |                                                                                              |
| Remove NaN          | Filter                    | Filters out articles with invalid publication dates| Merged RSS                     | Sort List                 |                                                                                              |
| Sort List           | Sort                      | Sorts articles by newest first                      | Remove NaN                     | Clean List                |                                                                                              |
| Clean List          | Code                      | Deduplicates, limits to top 50, formats for AI     | Sort List                      | ChatGPT 4o                | # Data Processing Filters, sorts and prepares news articles for AI selection                 |
| ChatGPT 4o          | OpenAI (LangChain)        | Selects top 5 relevant articles via GPT-4o         | Clean List                     | Split Out                 | # AI Selection Uses GPT-4o to select the top 5 most relevant articles for a senior executive |
| Split Out           | Split Out                 | Splits selected articles into individual items     | ChatGPT 4o                    | Fetch HTML                |                                                                                              |
| Fetch HTML          | HTTP Request              | Downloads full HTML of each selected article       | Split Out                     | Extract Text              |                                                                                              |
| Extract Text        | HTML                      | Extracts summary text snippets from HTML           | Fetch HTML                    | Clean Text                |                                                                                              |
| Clean Text          | Set                       | Combines extracted text into article summaries     | Extract Text                  | Aggregate                 |                                                                                              |
| Aggregate           | Aggregate                 | Combines all summaries into one JSON array         | Clean Text                    | Create HTML               | # Email Generation Creates and sends formatted HTML digest email with selected articles      |
| Create HTML         | HTML                      | Builds styled RTL HTML email body                   | Aggregate                    | Merge                     |                                                                                              |
| Get Date            | Function                  | Generates current date in Israel timezone           | Schedule Trigger              | Merge                     |                                                                                              |
| Merge               | Merge                     | Combines HTML content and date for email sending   | Create HTML, Get Date         | Send Daily News           |                                                                                              |
| Send Daily News     | Email Send                | Sends the final newsletter email via SMTP           | Merge                        | None                      |                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 20:00 (8 PM) Israel time.

2. **Create two RSS Feed Read nodes:**  
   - **RSS - Mako:** URL: `https://storage.googleapis.com/mako-sitemaps/rss-hp.xml`  
   - **RSS - Calcalist:** URL: `https://www.calcalist.co.il/GeneralRSS/0,16335,L-8,00.xml`  
   - Connect both to Schedule Trigger output.

3. **Create two Set nodes to clean RSS fields:**  
   - **Edit Fields - Mako:**  
     - Title: Remove “[PACK]” and brackets, trim whitespace.  
     - Link: Replace `/torrent/download/(\d+)\..*` with `/torrents/$1`.  
     - pubDate: Convert to Unix timestamp (milliseconds).  
   - **Edit Fields - Calcalist:**  
     - Title: Remove “[PACK] ” prefix and “1080p” suffix, append size extracted from content HTML.  
     - Link: Same replacement as Mako.  
     - pubDate: Convert to Unix timestamp.  
   - Connect RSS - Mako to Edit Fields - Mako; RSS - Calcalist to Edit Fields - Calcalist.

4. **Create Merge node (Merged RSS):**  
   - Merge the outputs of both Edit Fields nodes.  
   - Connect both Edit Fields nodes to this Merge node.

5. **Create Filter node (Remove NaN):**  
   - Filter out items where `pubDate` is NaN or invalid.  
   - Connect Merged RSS to Remove NaN.

6. **Create Sort node (Sort List):**  
   - Sort by `pubDate` descending.  
   - Connect Remove NaN to Sort List.

7. **Create Code node (Clean List):**  
   - JavaScript code to:  
     - Deduplicate articles by link.  
     - Take top 50 articles.  
     - Format numbered list of titles and links separated by double newlines.  
   - Connect Sort List to Clean List.

8. **Create OpenAI node (ChatGPT 4o):**  
   - Model: gpt-4o  
   - System prompt: instruct to select 5 most important articles for senior CEO, including at least one on current affairs/security.  
   - Input: Use `chatgpt_input` from Clean List.  
   - Output: JSON array with `article` and `link`.  
   - Connect Clean List to ChatGPT 4o.  
   - Set OpenAI API credentials.

9. **Create Split Out node:**  
   - Split the JSON array output from ChatGPT 4o into individual items.  
   - Connect ChatGPT 4o to Split Out.

10. **Create HTTP Request node (Fetch HTML):**  
    - Fetch full HTML content of each article using `link` field.  
    - Connect Split Out to Fetch HTML.

11. **Create HTML node (Extract Text):**  
    - Extract text using CSS selectors:  
      - `.calcalistArticleHeader .subTitle` → `data-calcalist`  
      - `.article-header header h2` → `data-mako`  
    - Connect Fetch HTML to Extract Text.

12. **Create Set node (Clean Text):**  
    - Assign:  
      - `title` from Split Out’s original article title.  
      - `summary` concatenating extracted `data-calcalist` and `data-mako`.  
      - `url` from original article link.  
    - Connect Extract Text to Clean Text.

13. **Create Aggregate node:**  
    - Aggregate all article summaries into a single JSON array.  
    - Connect Clean Text to Aggregate.

14. **Create HTML node (Create HTML):**  
    - Build full HTML email with inline CSS, RTL direction, and placeholders for article data (titles, summaries, URLs).  
    - Connect Aggregate to Create HTML.

15. **Create Function node (Get Date):**  
    - Generate current date formatted as DD/MM/YYYY in Asia/Jerusalem timezone.  
    - Connect Schedule Trigger to Get Date (parallel to RSS nodes).

16. **Create Merge node:**  
    - Combine Create HTML output and Get Date output into one JSON object.  
    - Connect Create HTML and Get Date to this Merge node.

17. **Create Email Send node (Send Daily News):**  
    - Configure SMTP credentials (host, port, username, password).  
    - Set email subject to include date (e.g., "סקירה ה-AI היומית שלך: {{ $json.date_today }}").  
    - Set recipient and sender emails.  
    - Connect Merge node to Send Daily News.

18. **Activate the workflow.**

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| The workflow uses Hebrew language with right-to-left (RTL) layout for proper display in emails.           | Important for correct rendering in email clients supporting RTL languages.                         |
| OpenAI GPT-4o model is used for advanced natural language understanding and selection of relevant articles.| Requires valid OpenAI API key with access to GPT-4o.                                              |
| SMTP credentials must be configured with app-specific passwords or OAuth2 tokens for secure email sending.| Example SMTP host: smtp.gmail.com, port 465 or 587.                                               |
| Customization tips include changing RSS sources, modifying GPT prompts, and adding integrations (Notion, Telegram).| Allows tailoring the digest for different sectors or distribution channels.                        |
| The HTML email template uses inline CSS and Google Fonts (Heebo, Assistant) for Hebrew typography.         | Ensures consistent styling across email clients.                                                  |
| Workflow designed to run once daily; ensure n8n instance timezone matches or adjust schedule accordingly.  | Avoids duplicate or missed runs.                                                                   |
| For troubleshooting, monitor API rate limits, RSS feed availability, and SMTP server logs.                 | Common points of failure in production.                                                           |

---

This documentation provides a complete, structured reference to understand, reproduce, and maintain the "Create Daily Israeli Economic Newsletter using RSS and GPT-4o" workflow in n8n.