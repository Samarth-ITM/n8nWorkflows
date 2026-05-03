Daily Tech News Digest from Google News Summarized with Llama AI and Email Delivery

https://n8nworkflows.xyz/workflows/daily-tech-news-digest-from-google-news-summarized-with-llama-ai-and-email-delivery-7386


# Daily Tech News Digest from Google News Summarized with Llama AI and Email Delivery

---

### 1. Workflow Overview

This workflow automates the daily collection, summarization, and email delivery of technology news sourced from Google News (India). It is designed for tech professionals, entrepreneurs, and enthusiasts who want a concise, AI-generated digest of the latest tech developments every morning at 8 AM IST.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Trigger & Data Fetching:** Automatically triggers daily at 8 AM and retrieves raw HTML tech news data from Google News India.
- **1.2 HTML Content Extraction & Formatting:** Parses the HTML to extract key news elements (titles, sources, publication times, snippets), formats this data into a structured summary, and counts articles.
- **1.3 Conditional Processing:** Checks if any news articles were found and routes workflow accordingly.
- **1.4 AI Analysis & Summarization:** Uses a Langchain agent powered by Llama AI to analyze the extracted news content and generate a professional daily summary.
- **1.5 Email Notification:** Sends the AI-generated tech news summary via email to the configured recipient.
- **1.6 Error Handling:** Sends an alert email if no news articles are found, indicating potential issues with data extraction or connectivity.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Data Fetching

**Overview:**  
This block initiates the workflow daily at 8 AM IST and fetches raw HTML content from Google News India’s tech news section.

**Nodes Involved:**  
- Schedule Daily Tech News Trigger  
- Fetch Google Tech News  

**Node Details:**  

- **Schedule Daily Tech News Trigger**  
  - Type: Schedule Trigger  
  - Configuration: Triggers once daily at 8:00 AM local time (India Standard Time)  
  - Inputs: None (start node)  
  - Outputs: Connects to the HTTP request node  
  - Edge Cases: Execution failures if n8n instance is down or system clock issues; misconfiguration of trigger time zone  
  - Version: 1.2  

- **Fetch Google Tech News**  
  - Type: HTTP Request  
  - Role: Fetches the HTML page from Google News search results for "tech news" filtered for India locale (`hl=en-IN&gl=IN&ceid=IN:en`)  
  - Configuration: GET request to URL with 15-second timeout to avoid hanging  
  - Inputs: Trigger node output  
  - Outputs: Raw HTML content passed to the HTML extraction node  
  - Edge Cases: Network failures, HTTP errors (e.g., 403 if Google blocks automated requests), timeout errors  
  - Version: 4.2  

---

#### 2.2 HTML Content Extraction & Formatting

**Overview:**  
Parses the fetched HTML content to extract key news details such as article titles, links, sources, publication times, and snippets. Then formats this extracted data into a readable summary string and counts the number of articles extracted.

**Nodes Involved:**  
- Extract Tech News Articles  
- Format Tech News Data  

**Node Details:**  

- **Extract Tech News Articles**  
  - Type: HTML Extract  
  - Role: Uses CSS selectors to parse the Google News HTML structure and extract arrays of data points: titles, links, sources, times, snippets, and entire article blocks.  
  - Configuration:  
    - Clean-up text enabled to remove unnecessary whitespace  
    - Selectors updated to match current Google News DOM structure (`article h3 a, article h4 a, .JtKRv` for titles/links, etc.)  
    - Returns multiple arrays for different data fields  
  - Inputs: Raw HTML from HTTP Request node  
  - Outputs: JSON object with extracted arrays for further processing  
  - Edge Cases: Changes in Google News DOM can break selectors; empty arrays if no articles found; partial extraction if network issue or HTML incomplete  
  - Version: 1.2  

- **Format Tech News Data**  
  - Type: Set  
  - Role: Constructs a formatted string summarizing the extracted data for AI input and email summary, and calculates the article count.  
  - Configuration:  
    - Uses JavaScript expressions to format current date/time and slice arrays to limit to top headlines (up to 15 titles, 10 times, 8 full articles)  
    - Includes extraction summary counts for validation  
  - Inputs: Extracted article data JSON  
  - Outputs: JSON with `Formatted_Tech_News` string and `Article_Count` number field  
  - Edge Cases: Empty or missing fields handled by fallback strings ("No titles found" etc.)  
  - Version: 3.4  

---

#### 2.3 Conditional Processing

**Overview:**  
Validates if any news articles were found before proceeding. If no articles, triggers error handling; otherwise, continues with AI processing.

**Nodes Involved:**  
- Check if News Found  

**Node Details:**  

- **Check if News Found**  
  - Type: If  
  - Role: Checks if `Article_Count` is greater than or equal to 1  
  - Configuration:  
    - Condition compares numeric value of `Article_Count` (from Format node) against 1  
  - Inputs: Formatted data with article count  
  - Outputs:  
    - True branch: proceeds to AI analysis node  
    - False branch: routes to error email alert node  
  - Edge Cases: Fails if `Article_Count` missing or not a number; misrouting if condition misconfigured  
  - Version: 2  

---

#### 2.4 AI Analysis & Summarization

**Overview:**  
Uses a Langchain agent powered by Llama AI to analyze the formatted news text and produce a concise, structured daily summary for tech professionals.

**Nodes Involved:**  
- AI Tech News Analyzer  
- LLM - Tech News Model  

**Node Details:**  

- **LLM - Tech News Model**  
  - Type: Langchain LLM Chat Ollama  
  - Role: Provides the Llama 3.2 16k token model for AI processing  
  - Configuration:  
    - Model: `llama3.2-16000:latest`  
    - Temperature: 0.3 for balanced creativity and relevance  
  - Credentials: Ollama API credentials required  
  - Inputs: Receives prompt from AI Tech News Analyzer node via `ai_languageModel` input  
  - Outputs: AI-generated text passed back to agent node  
  - Edge Cases: API connectivity issues, model unavailability, rate limiting, credential errors  
  - Version: 1  

- **AI Tech News Analyzer**  
  - Type: Langchain Agent  
  - Role: Defines the prompt and system instructions for AI summarization  
  - Configuration:  
    - Text prompt dynamically injects formatted tech news string from previous node  
    - System message guides LLM to produce a professional summary with six clearly defined sections (Key Trends, Major Announcements, Industry Impact, Emerging Tech, Market Movements, Outlook)  
    - Word limit: 300-400 words, markdown formatted with headers  
  - Inputs: Formatted news data from previous node and LLM model node via languageModel input  
  - Outputs: AI summary text to email node  
  - Edge Cases: Prompt generation errors, output formatting issues  
  - Version: 1.6  

---

#### 2.5 Email Notification

**Overview:**  
Sends the AI-generated daily tech news summary as a plain text email to a configured recipient.

**Nodes Involved:**  
- Send Tech News Email  

**Node Details:**  

- **Send Tech News Email**  
  - Type: Email Send  
  - Role: Sends the summarized news report via SMTP email  
  - Configuration:  
    - Subject line includes date dynamically formatted for India locale  
    - Body text set to AI summary output (`$json.output`)  
    - Recipient and sender emails set statically (replaceable placeholders)  
    - Email format: plain text  
  - Credentials: SMTP credentials required, verified with test SMTP config  
  - Inputs: AI summary text from Langchain agent node  
  - Outputs: None (end node)  
  - Edge Cases: SMTP authentication errors, email delivery failure, invalid email addresses  
  - Version: 2.1  

---

#### 2.6 Error Handling

**Overview:**  
If no articles are found, sends an alert email describing possible causes and requesting workflow review.

**Nodes Involved:**  
- Send Error Alert  

**Node Details:**  

- **Send Error Alert**  
  - Type: Email Send  
  - Role: Notifies recipient that no news articles were extracted, highlighting possible reasons  
  - Configuration:  
    - Static alert message with timestamp and troubleshooting hints (e.g., CSS selector changes, network issues)  
    - Subject prefixed with warning symbol  
    - Recipient and sender emails same as main email node (replaceable)  
    - Email format: plain text  
  - Credentials: Shares SMTP credentials with main email node  
  - Inputs: Triggered by false branch of the condition node  
  - Outputs: None (end node)  
  - Edge Cases: Same as email node (authentication and delivery issues)  
  - Version: 2.1  

---

### 3. Summary Table

| Node Name                  | Node Type                      | Functional Role                          | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                                                |
|----------------------------|--------------------------------|----------------------------------------|-------------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------------------------------------|
| Schedule Daily Tech News Trigger | Schedule Trigger               | Initiates workflow daily at 8 AM       | None                          | Fetch Google Tech News       |                                                                                                                                            |
| Fetch Google Tech News      | HTTP Request                   | Fetches tech news HTML from Google News India | Schedule Daily Tech News Trigger | Extract Tech News Articles  | Fetches tech news from Google News India                                                                                                   |
| Extract Tech News Articles  | HTML Extract                   | Parses HTML to extract titles, sources, times, snippets | Fetch Google Tech News          | Format Tech News Data        | Updated selectors for Google News structure - extracts titles, sources, times and full content                                             |
| Format Tech News Data       | Set                            | Formats extracted data into summary string and counts articles | Extract Tech News Articles      | Check if News Found          |                                                                                                                                            |
| Check if News Found         | If                             | Checks if any articles were found      | Format Tech News Data           | AI Tech News Analyzer / Send Error Alert |                                                                                                                                            |
| AI Tech News Analyzer       | Langchain Agent                | Generates AI summary from formatted news | Check if News Found (true)      | Send Tech News Email         |                                                                                                                                            |
| LLM - Tech News Model       | Langchain LLM Chat Ollama      | Provides Llama AI model for summarization | AI Tech News Analyzer (ai_languageModel) | AI Tech News Analyzer       |                                                                                                                                            |
| Send Tech News Email        | Email Send                     | Sends summarized news report via email | AI Tech News Analyzer           | None                        |                                                                                                                                            |
| Send Error Alert            | Email Send                     | Sends alert email if no news found     | Check if News Found (false)     | None                        |                                                                                                                                            |
| Workflow Info               | Sticky Note                   | Describes workflow purpose and use cases | None                          | None                        | 🚀 **Daily Tech News Automation**<br>• Scrapes Google News tech section daily at 8 AM<br>• Extracts headlines, sources & timestamps<br>• Uses AI to analyze trends and create summary<br>• Sends formatted email report<br>**Perfect for:** Tech professionals, startup founders, product managers, tech investors<br>**Data Sources:** Google News India (Tech), AI-powered analysis, Automated email delivery |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 8:00 AM (local time, IST)  
   - No credentials needed  
   - Position: Start node  

2. **Create HTTP Request Node ("Fetch Google Tech News")**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://news.google.com/search?q=tech%20news&hl=en-IN&gl=IN&ceid=IN%3Aen`  
   - Timeout: 15000 ms (15 seconds)  
   - Connect Schedule Trigger output to this node’s input  

3. **Create HTML Extract Node ("Extract Tech News Articles")**  
   - Type: HTML Extract  
   - Operation: Extract HTML Content  
   - Enable Cleanup Text  
   - Extraction Values:  
     - Article Titles: CSS selector `article h3 a, article h4 a, .JtKRv`, return array  
     - Article Links: Same selector as titles, return array  
     - Article Sources: CSS selector `article .wEwyrc, .vr1PYe, .CEMjEf`, return array  
     - Publication Times: CSS selector `article time, .WW6dff, .hvbAAd`, return array  
     - Article Snippets: CSS selector `article .GI74Re, .St8ea, .y3G2Ed`, return array  
     - All Articles: CSS selector `article`, return array  
   - Connect HTTP Request output to this node  

4. **Create Set Node ("Format Tech News Data")**  
   - Type: Set  
   - Add two fields:  
     - `Formatted_Tech_News` (string) with value:  
       ```
       DAILY TECH NEWS SUMMARY - {{ new Date().toLocaleDateString('en-IN') }}
       ========================================================

       Source: Google News - Tech Category (India)
       Extracted at: {{ new Date().toLocaleTimeString('en-IN') }}

       📰 HEADLINES EXTRACTED:
       {{ $json['Article Titles'] && Array.isArray($json['Article Titles']) ? $json['Article Titles'].slice(0, 15).map((title, i) => `${i+1}. ${title}`).join('\n') : 'No titles found' }}

       ⏰ PUBLICATION TIMES:
       {{ $json['Publication Times'] && Array.isArray($json['Publication Times']) ? $json['Publication Times'].slice(0, 10).join(' | ') : 'No times found' }}

       📄 ARTICLE CONTENT FOR AI ANALYSIS:
       {{ $json['All Articles'] && Array.isArray($json['All Articles']) ? $json['All Articles'].slice(0, 8).join('\n\n=== NEXT ARTICLE ===\n\n') : 'No article content found' }}

       📊 EXTRACTION SUMMARY:
       Titles found: {{ $json['Article Titles'] ? $json['Article Titles'].length : 0 }}
       Sources found: {{ $json['Article Sources'] ? $json['Article Sources'].length : 0 }}
       Times found: {{ $json['Publication Times'] ? $json['Publication Times'].length : 0 }}
       ```
     - `Article_Count` (number): `={{ $json['Article Titles'].length }}`  
   - Connect Extract Tech News Articles output to this node  

5. **Create If Node ("Check if News Found")**  
   - Type: If  
   - Condition: `Article_Count >= 1` (number comparison)  
   - Connect Format Tech News Data output to this node  

6. **Create Langchain LLM Node ("LLM - Tech News Model")**  
   - Type: Langchain LLM Chat Ollama  
   - Model: `llama3.2-16000:latest`  
   - Temperature: 0.3  
   - Configure Ollama API credentials  
   - No direct input connection (used as AI language model reference)  

7. **Create Langchain Agent Node ("AI Tech News Analyzer")**  
   - Type: Langchain Agent  
   - Text prompt:  
     ```
     Analyze and summarize today's tech news:

     {{ $json['Formatted_Tech_News'] }}
     ```
   - System message instructing the model to create a professional summary with sections: Key Tech Trends, Major Announcements, Industry Impact, Emerging Technologies, Market Movements, Outlook; limit 300-400 words, markdown formatting  
   - Link `ai_languageModel` input to the LLM node  
   - Connect If node’s true output to this node  

8. **Create Email Send Node ("Send Tech News Email")**  
   - Type: Email Send  
   - Subject: `🚀 Daily Tech News Summary - {{ new Date().toLocaleDateString('en-IN') }}`  
   - To Email: `recipient@gmail.com` (replace with actual recipient)  
   - From Email: `your-email@gmail.com` (replace with actual sender)  
   - Text: `={{ $json.output }}` (content from AI Tech News Analyzer)  
   - Email format: Text  
   - Configure SMTP credentials (use tested SMTP server)  
   - Connect AI Tech News Analyzer output to this node  

9. **Create Email Send Node ("Send Error Alert")**  
   - Type: Email Send  
   - Subject: `⚠️ Tech News Workflow Alert - No Articles Found`  
   - To Email: same as above  
   - From Email: same as above  
   - Text:  
     ```
     The daily tech news workflow ran but found no articles to process.

     This could be due to:
     - Changes in Google News website structure
     - Network connectivity issues
     - CSS selector updates needed

     Please check the workflow configuration.

     Time: {{ new Date().toLocaleString('en-IN') }}
     ```
   - Email format: Text  
   - Use same SMTP credentials as above  
   - Connect If node’s false output to this node  

10. **Add Sticky Note ("Workflow Info")**  
    - Content describing workflow purpose, audience, data sources, and summary of automation steps for documentation clarity  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                              | Context or Link                                                 |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------|
| 🚀 **Daily Tech News Automation**: Scrapes Google News tech section daily at 8 AM, extracts headlines, sources & timestamps, uses AI for summary, emails report | Workflow purpose and overview sticky note                       |
| CSS selectors for extraction may need periodic updates if Google News changes its website structure                                                      | Maintenance note                                                |
| Uses Ollama API with Llama 3.2 (16k tokens) for AI summarization                                                                                          | LLM model technology detail                                    |
| SMTP credentials must be valid and tested to ensure email delivery                                                                                        | Email sending prerequisite                                     |
| AI summary prompt limits output to 300-400 words with markdown headers to maintain readability and structure                                             | Prompt design best practice                                    |

---

**Disclaimer:** The provided text is exclusively derived from an n8n automated workflow. It strictly complies with content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.