Scrape recent news about a company before a call

https://n8nworkflows.xyz/workflows/scrape-recent-news-about-a-company-before-a-call-2110


# Scrape recent news about a company before a call

### 1. Workflow Overview

This workflow is designed to automate the preparation for sales or business calls by delivering the latest news about companies you are scheduled to meet with. Its main goal is to save time and enhance meeting readiness by eliminating manual research, providing curated, up-to-date news directly via email.

**Target Use Cases:**  
- Sales professionals preparing for calls or meetings  
- Business development teams needing quick company insights  
- Customer success managers who want to stay informed about clients

**Logical Blocks:**  
- **1.1 Trigger & Setup:** Scheduled daily execution and initial parameter configuration  
- **1.2 Calendar Event Retrieval:** Fetching today’s meetings from Google Calendar  
- **1.3 Meeting Filtering & Company Extraction:** Selecting relevant meetings and extracting company names  
- **1.4 News Retrieval:** Querying newsapi.org for recent news articles about each company  
- **1.5 Email Formatting & Delivery:** Formatting news items into HTML and sending personalized emails via Gmail

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Setup

**Overview:**  
This block triggers the workflow every morning at 7 AM and sets up essential parameters including API keys, email recipients, and news search constraints.

**Nodes Involved:**  
- Every morning @ 7 (Schedule Trigger)  
- Setup (Set node)  
- Sticky Note (Instructional content)  
- Sticky Note1 (Configuration instructions)

**Node Details:**  

- **Every morning @ 7**  
  - Type: Schedule Trigger  
  - Role: Initiates the workflow daily at 7:00 AM  
  - Configuration: Trigger at hour 7 every day  
  - Inputs: None (start node)  
  - Outputs: Connects to Setup node  
  - Edge cases: Missed trigger if workflow or server down; timezone considerations  

- **Setup**  
  - Type: Set  
  - Role: Defines workflow parameters for use downstream  
  - Configuration:  
    - `apiKey`: API key for NewsAPI (hardcoded example key)  
    - `newsAge`: Defines how many days old news can be (default 10)  
    - `maxArticles`: Maximum number of articles per company (default 20)  
    - `emails`: Recipient emails (empty by default, must be configured)  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: Connects to Get meetings for today node  
  - Edge cases: Missing or invalid API key causes HTTP request failures; empty emails list leads to no email sent  

- **Sticky Notes**  
  - Provide user guidance on setup and workflow purpose  
  - No functional impact but critical for user understanding and configuration  

---

#### 1.2 Calendar Event Retrieval

**Overview:**  
This block queries Google Calendar for all meetings scheduled today.

**Nodes Involved:**  
- Get meetings for today (Google Calendar node)

**Node Details:**  

- **Get meetings for today**  
  - Type: Google Calendar  
  - Role: Retrieves all calendar events starting today and ending tomorrow (24-hour window)  
  - Configuration:  
    - `timeMin`: Today’s date  
    - `timeMax`: Tomorrow’s date  
    - `singleEvents`: True (expands recurring events)  
    - Calendar: Specific user’s calendar email configured  
  - Credentials: Google Calendar OAuth2 required  
  - Inputs: From Setup node  
  - Outputs: Connects to Filter meetings node  
  - Edge cases: Authentication failure; empty calendar returns no data; timezone mismatch could cause missing events  

---

#### 1.3 Meeting Filtering & Company Extraction

**Overview:**  
Filters meetings to only those starting with "call with" or "meeting with" (case-insensitive), then extracts the company name from the event summary.

**Nodes Involved:**  
- Filter meetings (If node)  
- Extract company name (Set node)  
- No meetings today (NoOp node)  
- Sticky Note2 (Instructional content)

**Node Details:**  

- **Filter meetings**  
  - Type: If  
  - Role: Checks if event summary starts with "call with" or "meeting with" (lowercase)  
  - Configuration: Two OR conditions using string startsWith operator  
  - Inputs: From Get meetings for today  
  - Outputs:  
    - True branch: Extract company name  
    - False branch: No meetings today  
  - Edge cases: Event summaries not following naming convention are skipped; case sensitivity handled by lowercasing  

- **Extract company name**  
  - Type: Set  
  - Role: Derives the company name by removing the prefixes "meeting with" or "call with" from the event summary and trimming whitespace  
  - Configuration: Uses JavaScript expression to transform summary text to lowercase, strip phrases, and trim  
  - Inputs: From Filter meetings (true branch)  
  - Outputs: Connects to Get latest news node  
  - Edge cases: Irregular event summary formats may produce incorrect company names  

- **No meetings today**  
  - Type: NoOp  
  - Role: Placeholder node for cases where no relevant meetings are present  
  - Inputs: From Filter meetings (false branch)  
  - Outputs: None  
  - Edge cases: Gracefully handles days with no meetings  

- **Sticky Note2**  
  - Description reminding user that filtering is based on summary prefixes and can be customized  

---

#### 1.4 News Retrieval

**Overview:**  
Searches newsapi.org for recent English-language news articles about each extracted company within the specified news age and maximum article count.

**Nodes Involved:**  
- Get latest news (HTTP Request)  
- Sticky Note (general workflow description)

**Node Details:**  

- **Get latest news**  
  - Type: HTTP Request  
  - Role: Queries newsapi.org’s `everything` endpoint for recent articles matching the company name  
  - Configuration:  
    - URL built dynamically using expressions:  
      - API key from Setup node  
      - Query parameter `q` set to company name  
      - `from` date calculated as current date minus newsAge days  
      - Sorted by `publishedAt`  
      - Language restricted to English  
      - Page size limited by maxArticles  
      - Search limited to article titles (`searchIn=title`)  
  - Inputs: From Extract company name  
  - Outputs: Connects to Format for email  
  - Edge cases: API key invalid or rate limits exceeded cause errors; company names with special characters may require encoding; empty results possible  

---

#### 1.5 Email Formatting & Delivery

**Overview:**  
Formats the news articles into an HTML email and sends personalized emails to configured recipients.

**Nodes Involved:**  
- Format for email (Code node)  
- Send news (Gmail node)

**Node Details:**  

- **Format for email**  
  - Type: Code (JavaScript)  
  - Role: Generates HTML table with articles, including clickable titles, descriptions, and sources  
  - Configuration:  
    - Loops over articles array  
    - For each article, creates a styled row with title linking to article URL, description or content snippet, and source name if available  
    - Returns compiled HTML string in `html` field  
  - Inputs: From Get latest news  
  - Outputs: Connects to Send news  
  - Edge cases: Articles without description fallback to content; missing source names handled gracefully; empty article lists produce empty table  

- **Send news**  
  - Type: Gmail  
  - Role: Sends the formatted email to the recipient list  
  - Configuration:  
    - `sendTo`: Email addresses from Setup node (comma-separated)  
    - `subject`: Dynamic, includes the original meeting summary (company)  
    - `message`: HTML content from Format for email node  
  - Credentials: Gmail OAuth2 account configured  
  - Inputs: From Format for email  
  - Outputs: None  
  - Edge cases: Authentication failures; invalid recipient emails; large emails may cause sending delays or truncation  

---

### 3. Summary Table

| Node Name             | Node Type            | Functional Role                              | Input Node(s)               | Output Node(s)             | Sticky Note                                                                                                  |
|-----------------------|----------------------|----------------------------------------------|-----------------------------|----------------------------|--------------------------------------------------------------------------------------------------------------|
| Every morning @ 7     | Schedule Trigger      | Triggers workflow daily at 7 AM              | None                        | Setup                      |                                                                                                              |
| Setup                 | Set                  | Sets API key, news age, max articles, emails| Every morning @ 7           | Get meetings for today     | Configuration instructions for API key, news age, max articles, recipient emails                              |
| Get meetings for today| Google Calendar      | Fetches today’s calendar events               | Setup                       | Filter meetings            |                                                                                                              |
| Filter meetings       | If                   | Filters meetings starting with "call with"/"meeting with" | Get meetings for today    | Extract company name / No meetings today | Filtering criteria explanation: customizable prefixes                                                        |
| Extract company name  | Set                  | Extracts company name from event summary      | Filter meetings (true branch)| Get latest news            |                                                                                                              |
| Get latest news       | HTTP Request          | Fetches recent news articles from NewsAPI     | Extract company name        | Format for email           |                                                                                                              |
| Format for email      | Code                  | Formats news articles into HTML email content| Get latest news             | Send news                  |                                                                                                              |
| Send news             | Gmail                 | Sends email with news to recipients           | Format for email            | None                       |                                                                                                              |
| No meetings today     | NoOp                  | Placeholder when no applicable meetings found| Filter meetings (false branch)| None                      |                                                                                                              |
| Sticky Note           | Sticky Note           | Describes workflow purpose                     | None                        | None                       | Latest company news before a call                                                                            |
| Sticky Note1          | Sticky Note           | Setup instructions                             | None                        | None                       | Configure your workflow here with API key, newsAge, maxArticles, emails                                      |
| Sticky Note2          | Sticky Note           | Filtering explanation                           | None                        | None                       | This will get all meetings starting with 'Meeting with' or 'Call with' but can be customized                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Name: `Every morning @ 7`  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 7:00 AM  

2. **Create Set node for parameters**  
   - Name: `Setup`  
   - Add fields:  
     - `apiKey`: Your NewsAPI API key (string)  
     - `newsAge`: Number of days for news age (number, e.g., 10)  
     - `maxArticles`: Max articles per company (string, e.g., "20")  
     - `emails`: Comma-separated emails for news delivery (string)  
   - Connect `Every morning @ 7` output to this node  

3. **Create Google Calendar node**  
   - Name: `Get meetings for today`  
   - Operation: Get All Events  
   - Options:  
     - `timeMin`: Expression `={{ $today }}` (today’s date)  
     - `timeMax`: Expression `={{ $today.plus({ days: 1 }) }}` (tomorrow’s date)  
     - `singleEvents`: True  
     - Calendar: Your Google Calendar email  
   - Set Google Calendar OAuth2 credentials  
   - Connect `Setup` output to this node  

4. **Create If node**  
   - Name: `Filter meetings`  
   - Conditions: OR of two string checks on event summary (lowercase) startsWith:  
     - "call with"  
     - "meeting with"  
   - Connect `Get meetings for today` output to this node  

5. **Create Set node**  
   - Name: `Extract company name`  
   - Field `companyName`: Expression to strip prefixes and trim:  
     ```
     ={{ $json.summary.toLowerCase().replace('meeting with', '').replace('call with', '').trim() }}
     ```  
   - Connect `Filter meetings` True output to this node  

6. **Create HTTP Request node**  
   - Name: `Get latest news`  
   - Method: GET  
   - URL expression:  
     ```
     =https://newsapi.org/v2/everything?apiKey={{ $('Setup').first().json.apiKey }}&q={{ $json.companyName }}&from={{ DateTime.now().minus({ days: $('Setup').first().json.newsAge }).toFormat('yyyy-MM-dd') }}&sortBy=publishedAt&language=en&pageSize={{ $('Setup').first().json.maxArticles }}&searchIn=title
     ```  
   - Connect `Extract company name` output to this node  

7. **Create Code node**  
   - Name: `Format for email`  
   - Mode: Run Once For Each Item  
   - JavaScript code:  
     ```js
     let html = `<table style="width: 100%">`;
     for (const article of $input.item.json.articles) {
       html += `
         <tr>
           <td style="display: flex; background-color: #f2f4f8; font-family: sans-serif; padding: 0.3em 0.5em">
             <div style="padding: 1em">
               <a style="display: block; margin-bottom: 10px; font-size: 1.2em" href="${article.url}">${article.title}</a>
               <i>
                 ${article.description ? article.description : article.content}
               </i>
               <div style="margin-top: 1em">
                 ${article.source?.name ? '<b>Source:</b> ' + article.source.name : ''}
               </div>
             </div>
           </td>
         </tr>`;
     }
     html += '</table>';
     return { html };
     ```  
   - Connect `Get latest news` output to this node  

8. **Create Gmail node**  
   - Name: `Send news`  
   - Credentials: Gmail OAuth2 account configured  
   - Parameters:  
     - Send To: Expression `={{ $('Setup').first().json.emails }}`  
     - Subject: Expression `=Latest news for '{{ $('Extract company name').item.json.summary }}'`  
     - Message: Expression `={{ $json.html }}`  
   - Connect `Format for email` output to this node  

9. **Create NoOp node**  
   - Name: `No meetings today`  
   - Connect `Filter meetings` False output to this node  

10. **Optionally add Sticky Notes** at relevant positions for user guidance and instructions.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------|
| Workflow purpose: Automate delivery of recent company news before sales calls to improve preparedness and save research time.              | Workflow description                             |
| News API requires a valid API key: https://newsapi.org                                                                                      | Setup node configuration                         |
| Gmail node requires OAuth2 credentials with send email permissions                                                                          | Gmail node credential setup                       |
| Google Calendar node requires OAuth2 credentials with calendar read permissions                                                             | Google Calendar credential setup                  |
| Filtering on event summaries can be customized to match your own meeting naming conventions                                                  | Sticky Note2 content                              |
| This workflow can be adapted to other calendar systems or news providers by modifying nodes accordingly                                      | General adaptation note                           |
| Video/visual setup instructions provided in workflow description image: https://i.imgur.com/AvStsGb.png                                     | Workflow setup illustration                       |

---

This structured reference document fully captures the workflow’s logic, configuration, and edge cases, enabling precise understanding, modification, or reconstruction.