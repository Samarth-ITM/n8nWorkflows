Automated LinkedIn Lead Generation, Scoring & Communication with AI-Agent

https://n8nworkflows.xyz/workflows/automated-linkedin-lead-generation--scoring---communication-with-ai-agent-3490


# Automated LinkedIn Lead Generation, Scoring & Communication with AI-Agent

### 1. Workflow Overview

This workflow automates LinkedIn lead generation, enrichment, scoring, and communication using AI and the HDW LinkedIn community node on a self-hosted n8n instance. It transforms an Ideal Customer Profile (ICP) description into LinkedIn Sales Navigator search filters, discovers leads, enriches their data with company websites, posts, and news, scores leads based on intent signals, and automates connection requests and follow-up messaging.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception & ICP Translation**: Receives ICP input via chat and converts it into LinkedIn search parameters using an AI agent.
- **1.2 Lead Discovery & Initial Data Storage**: Searches LinkedIn for leads matching the filters and stores results in Google Sheets.
- **1.3 Company Website Enrichment**: Retrieves and summarizes company websites for leads.
- **1.4 Lead & Company Content Enrichment**: Collects and summarizes LinkedIn posts by leads and companies, and company news.
- **1.5 Lead Scoring**: Uses AI to score leads based on enriched data.
- **1.6 Connection Requests & Status Tracking**: Sends connection requests to top leads and tracks connection status.
- **1.7 Messaging Automation**: Sends personalized messages to connected leads.
- **1.8 Scheduling & Rate Limiting**: Controls timing and volume of connection requests and messages to comply with LinkedIn limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception & ICP Translation

**Overview:**  
This block receives the ICP description from a chat interface and uses an AI agent to convert it into structured LinkedIn Sales Navigator search filters.

**Nodes Involved:**  
- When chat message received  
- OpenAI Chat Model  
- Simple Memory  
- Structured Output Parser  
- AI Agent: ICP -> LinkedIn search filters  
- Split Out

**Node Details:**

- **When chat message received**  
  - Type: LangChain Chat Trigger  
  - Role: Entry point for ICP input via chat  
  - Config: Default webhook, no special options  
  - Input: External chat message  
  - Output: Passes chat content to AI agent  
  - Edge cases: Missing or malformed input; webhook failures

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Processes chat input with GPT-4o  
  - Config: Model set to "gpt-4o"  
  - Credentials: OpenAI API key required  
  - Edge cases: API rate limits, network errors

- **Simple Memory**  
  - Type: LangChain Memory Buffer Window  
  - Role: Maintains conversational context for chat  
  - Config: Default buffer window  
  - Edge cases: Memory overflow or loss

- **Structured Output Parser**  
  - Type: LangChain Structured Output Parser  
  - Role: Parses AI output into JSON matching LinkedIn search parameters  
  - Config: Uses a JSON schema example for Sales Navigator filters  
  - Edge cases: Parsing errors if AI output is malformed

- **AI Agent: ICP -> LinkedIn search filters**  
  - Type: LangChain Agent  
  - Role: Converts ICP text into LinkedIn Sales Navigator search filters  
  - Config: System prompt includes detailed instructions and a comprehensive industry list to standardize output  
  - Output: JSON array of search parameter objects with counts  
  - Edge cases: Incomplete or ambiguous ICP descriptions; invalid JSON output

- **Split Out**  
  - Type: Split Out  
  - Role: Splits array of search filters into individual items for parallel processing  
  - Edge cases: Empty arrays or unexpected data types

---

#### 1.2 Lead Discovery & Initial Data Storage

**Overview:**  
Uses the HDW LinkedIn node to search LinkedIn with the AI-generated filters, then saves lead data to Google Sheets.

**Nodes Involved:**  
- HDW LinkedIn SN  
- Loop Over Items5  
- Google Sheets  
- Google Sheets1  
- Company name is not empty (If)  
- Loop Over Items

**Node Details:**

- **HDW LinkedIn SN**  
  - Type: HDW LinkedIn Community Node  
  - Role: Searches LinkedIn Sales Navigator with parameters from AI agent  
  - Config: Uses dynamic parameters from AI output, including keywords, titles, companies, location, industry, company sizes  
  - Credentials: HDW API key required  
  - Edge cases: API limits, invalid filters, network errors

- **Loop Over Items5**  
  - Type: Split In Batches  
  - Role: Processes each lead item individually for subsequent enrichment  
  - Config: Default batch size, no reset  
  - Edge cases: Batch size too large causing timeouts

- **Google Sheets**  
  - Type: Google Sheets Append or Update  
  - Role: Saves lead data to Google Sheets with columns like Name, URN, URL, Headline, Location, Company, Industry, etc.  
  - Credentials: Google Sheets OAuth2  
  - Edge cases: Sheet access errors, rate limits

- **Google Sheets1**  
  - Type: Google Sheets Read  
  - Role: Reads lead data for further processing  
  - Credentials: Google Sheets OAuth2  
  - Edge cases: Empty or corrupted sheet data

- **Company name is not empty (If)**  
  - Type: If  
  - Role: Checks if lead's current company name exists and website is empty to trigger website enrichment  
  - Edge cases: Missing company names or websites

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Processes leads with company names for website enrichment  
  - Edge cases: Large batches causing delays

---

#### 1.3 Company Website Enrichment

**Overview:**  
Retrieves company websites via Google search and summarizes website content using AI.

**Nodes Involved:**  
- HDW Get Company Website  
- Google Sheets2  
- Google Sheets3  
- Website is not empty (If)  
- Loop Over Items1  
- HDW Site-map  
- HDW Parser  
- Summarise company website  
- Google Sheets10

**Node Details:**

- **HDW Get Company Website**  
  - Type: HDW LinkedIn Node (Google Search)  
  - Role: Searches Google for company website using company name query  
  - Config: Query format: "{Company Name} company Website"  
  - Credentials: HDW API key  
  - Edge cases: No website found, ambiguous results

- **Google Sheets2**  
  - Type: Google Sheets Update  
  - Role: Updates lead row with found website URL  
  - Edge cases: Sheet update conflicts

- **Google Sheets3**  
  - Type: Google Sheets Read  
  - Role: Reads updated lead data for website enrichment  
  - Edge cases: Sheet read errors

- **Website is not empty (If)**  
  - Type: If  
  - Role: Checks if website URL exists and product summary is empty to trigger scraping and summarization  
  - Edge cases: Missing or invalid URLs

- **Loop Over Items1**  
  - Type: Split In Batches  
  - Role: Processes leads with websites for scraping  
  - Edge cases: Large batch size

- **HDW Site-map**  
  - Type: HDW Web Parser Tool  
  - Role: Retrieves sitemap from company website URL to identify relevant pages  
  - Edge cases: No sitemap found, site restrictions

- **HDW Parser**  
  - Type: HDW Web Parser Tool  
  - Role: Scrapes content from identified sitemap URLs  
  - Edge cases: Scraping blocked by robots.txt, timeouts

- **Summarise company website**  
  - Type: LangChain Agent with OpenAI  
  - Role: Summarizes scraped website content into business and product info  
  - Edge cases: AI API failures, incomplete data

- **Google Sheets10**  
  - Type: Google Sheets Update  
  - Role: Saves summarized product info back to Google Sheets  
  - Edge cases: Sheet update errors

---

#### 1.4 Lead & Company Content Enrichment

**Overview:**  
Collects and summarizes LinkedIn posts by leads and companies, and company news to enrich lead profiles.

**Nodes Involved:**  
- HDW Get User Posts  
- Aggregate  
- Summarise user posts  
- Google Sheets5  
- Post summary is empty (If)  
- Loop Over Items2  
- HDW Get Company News  
- Aggregate1  
- Summarise company news  
- Google Sheets7  
- Company news is empty (If)  
- Loop Over Items3  
- HDW Get Company Posts  
- Aggregate2  
- Summarise company posts  
- Google Sheets9  
- Company post is empty (If)  
- Loop Over Items4

**Node Details:**

- **HDW Get User Posts**  
  - Type: HDW LinkedIn Node  
  - Role: Retrieves LinkedIn posts of the lead  
  - Edge cases: No posts found, API limits

- **Aggregate**  
  - Type: Aggregate  
  - Role: Combines text and repost content from posts for summarization  
  - Edge cases: Empty post content

- **Summarise user posts**  
  - Type: OpenAI Node  
  - Role: Summarizes lead's posts into concise content  
  - Edge cases: API failures

- **Google Sheets5**  
  - Type: Google Sheets Update  
  - Role: Saves posts summary to lead row  
  - Edge cases: Sheet update conflicts

- **Post summary is empty (If)**  
  - Type: If  
  - Role: Checks if posts summary is missing to trigger fetching posts  
  - Edge cases: Missing URN or posts

- **Loop Over Items2**  
  - Type: Split In Batches  
  - Role: Processes leads missing posts summary  
  - Edge cases: Large batch size

- **HDW Get Company News**  
  - Type: HDW LinkedIn Node (Google Search)  
  - Role: Searches Google for recent news about the lead's company  
  - Edge cases: No news found

- **Aggregate1**  
  - Type: Aggregate  
  - Role: Aggregates news descriptions for summarization  
  - Edge cases: Empty news content

- **Summarise company news**  
  - Type: OpenAI Node  
  - Role: Summarizes company news content  
  - Edge cases: API failures

- **Google Sheets7**  
  - Type: Google Sheets Update  
  - Role: Saves company news summary  
  - Edge cases: Sheet update errors

- **Company news is empty (If)**  
  - Type: If  
  - Role: Checks if company news summary is missing  
  - Edge cases: Missing URN or news

- **Loop Over Items3**  
  - Type: Split In Batches  
  - Role: Processes leads missing company news summary  
  - Edge cases: Large batch size

- **HDW Get Company Posts**  
  - Type: HDW LinkedIn Node  
  - Role: Retrieves LinkedIn posts from the lead's company page  
  - Edge cases: No posts found

- **Aggregate2**  
  - Type: Aggregate  
  - Role: Aggregates company posts text for summarization  
  - Edge cases: Empty posts

- **Summarise company posts**  
  - Type: OpenAI Node  
  - Role: Summarizes company posts content  
  - Edge cases: API failures

- **Google Sheets9**  
  - Type: Google Sheets Update  
  - Role: Saves company posts summary  
  - Edge cases: Sheet update errors

- **Company post is empty (If)**  
  - Type: If  
  - Role: Checks if company post summary is missing  
  - Edge cases: Missing company URN or posts

- **Loop Over Items4**  
  - Type: Split In Batches  
  - Role: Processes leads missing company post summary  
  - Edge cases: Large batch size

---

#### 1.5 Lead Scoring

**Overview:**  
Uses AI to analyze enriched data and score leads on a 1-10 scale based on likelihood of interest.

**Nodes Involved:**  
- Lead Score is empty (If)  
- Loop Over Items6  
- Company Score Analysis  
- Google Sheets12  
- Google Sheets13  
- Sort

**Node Details:**

- **Lead Score is empty (If)**  
  - Type: If  
  - Role: Checks if lead score is missing to trigger scoring  
  - Edge cases: Missing data fields

- **Loop Over Items6**  
  - Type: Split In Batches  
  - Role: Processes leads missing scores  
  - Edge cases: Batch size and timeouts

- **Company Score Analysis**  
  - Type: OpenAI Node  
  - Role: Scores lead based on posts summary, company website summary, news, and posts  
  - Config: System prompt defines scoring criteria focused on mentions of hotel suppliers and services  
  - Output: Single numeric score 1-10  
  - Edge cases: API failures, ambiguous data

- **Google Sheets12**  
  - Type: Google Sheets Update  
  - Role: Saves lead score to sheet  
  - Edge cases: Sheet update conflicts

- **Google Sheets13**  
  - Type: Google Sheets Read  
  - Role: Reads scored leads for sorting and further processing  
  - Edge cases: Sheet read errors

- **Sort**  
  - Type: Sort  
  - Role: Sorts leads descending by Lead Score to prioritize outreach  
  - Edge cases: Missing or invalid scores

---

#### 1.6 Connection Requests & Status Tracking

**Overview:**  
Sends connection requests to top leads, tracks connection status, and updates Google Sheets accordingly.

**Nodes Involved:**  
- If2  
- Limit  
- Loop Over Items7  
- HDW Send LinkedIn Connection  
- Google Sheets14  
- Schedule Trigger  
- HDW Get LinkedIn Profile Connections  
- Split LinkedIn connections to items  
- Google Sheets15  
- Google Sheets16

**Node Details:**

- **If2**  
  - Type: If  
  - Role: Filters leads without existing contact requests to avoid duplicates  
  - Edge cases: Incorrect filtering logic

- **Limit**  
  - Type: Limit  
  - Role: Limits number of connection requests per run (max 20) to comply with LinkedIn limits (~100-200/week)  
  - Edge cases: Over-limit requests

- **Loop Over Items7**  
  - Type: Split In Batches  
  - Role: Processes limited leads for connection requests  
  - Edge cases: Batch size management

- **HDW Send LinkedIn Connection**  
  - Type: HDW LinkedIn Management Node  
  - Role: Sends connection request to lead URN  
  - Edge cases: API errors, rate limits, LinkedIn restrictions

- **Google Sheets14**  
  - Type: Google Sheets Update  
  - Role: Marks lead as having a contact request sent  
  - Edge cases: Sheet update errors

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow daily at 7 AM to send connection requests  
  - Edge cases: Scheduling conflicts

- **HDW Get LinkedIn Profile Connections**  
  - Type: HDW LinkedIn Management Node  
  - Role: Retrieves current LinkedIn connections to check for accepted requests  
  - Edge cases: API limits

- **Split LinkedIn connections to items**  
  - Type: Code Node  
  - Role: Converts array of connections into individual items for processing  
  - Edge cases: Empty or malformed data

- **Google Sheets15**  
  - Type: Google Sheets Update  
  - Role: Marks leads as connected in sheet  
  - Edge cases: Sheet update conflicts

- **Google Sheets16**  
  - Type: Google Sheets Read  
  - Role: Reads connected leads to prepare for messaging  
  - Edge cases: Sheet read errors

---

#### 1.7 Messaging Automation

**Overview:**  
Sends personalized messages to leads who have accepted connection requests and updates message status.

**Nodes Involved:**  
- Loop Over Items8  
- HDW LinkedIn Send Message  
- Google Sheets17  
- Schedule Trigger1

**Node Details:**

- **Loop Over Items8**  
  - Type: Split In Batches  
  - Role: Processes connected leads for messaging  
  - Edge cases: Batch size management

- **HDW LinkedIn Send Message**  
  - Type: HDW LinkedIn Management Node  
  - Role: Sends a personalized message to the lead's LinkedIn URN  
  - Config: Message text is customizable (default "Hello")  
  - Edge cases: API errors, message limits

- **Google Sheets17**  
  - Type: Google Sheets Update  
  - Role: Marks lead as having received a message  
  - Edge cases: Sheet update conflicts

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow daily at 7 AM to check connections and send messages  
  - Edge cases: Scheduling conflicts

---

#### 1.8 Scheduling & Rate Limiting

**Overview:**  
Controls timing of connection requests and messaging to comply with LinkedIn limits and optimize outreach.

**Nodes Involved:**  
- Schedule Trigger  
- Schedule Trigger1  
- Limit

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Runs daily at 7 AM to send connection requests  
  - Edge cases: Timezone misconfiguration

- **Schedule Trigger1**  
  - Type: Schedule Trigger  
  - Role: Runs daily at 7 AM to check connections and send messages  
  - Edge cases: Timezone misconfiguration

- **Limit**  
  - Type: Limit  
  - Role: Caps connection requests to max 20 per run (approx. 140 per week)  
  - Edge cases: Exceeding LinkedIn limits may cause account restrictions

---

### 3. Summary Table

| Node Name                      | Node Type                         | Functional Role                              | Input Node(s)                       | Output Node(s)                     | Sticky Note                                                                                               |
|--------------------------------|----------------------------------|----------------------------------------------|-----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------------------|
| When chat message received      | LangChain Chat Trigger           | Receives ICP input via chat                   | -                                 | OpenAI Chat Model                 |                                                                                                          |
| OpenAI Chat Model               | LangChain OpenAI Chat Model      | Processes chat input                          | When chat message received         | Simple Memory                    |                                                                                                          |
| Simple Memory                  | LangChain Memory Buffer          | Maintains chat context                        | OpenAI Chat Model                  | AI Agent: ICP -> LinkedIn search filters |                                                                                                          |
| Structured Output Parser        | LangChain Output Parser          | Parses AI output into JSON filters            | AI Agent: ICP -> LinkedIn search filters | AI Agent: ICP -> LinkedIn search filters | For changing limits use "count"                                                                           |
| AI Agent: ICP -> LinkedIn search filters | LangChain Agent                 | Converts ICP text to LinkedIn search filters | Simple Memory, OpenAI Chat Model   | Split Out                       | This node - an AI agent transforms your ICP description into filters for querying data in LinkedIn       |
| Split Out                      | Split Out                       | Splits array of filters into individual items | AI Agent: ICP -> LinkedIn search filters | HDW LinkedIn SN                 |                                                                                                          |
| HDW LinkedIn SN                | HDW LinkedIn Community Node      | Searches LinkedIn with filters                | Split Out                        | Loop Over Items5                | Data is requested through the HDW API for each filter and saved in Google Sheets.                         |
| Loop Over Items5               | Split In Batches                 | Processes each lead item                       | HDW LinkedIn SN                  | Google Sheets, HDW LinkedIn SN  |                                                                                                          |
| Google Sheets                 | Google Sheets Append/Update      | Saves lead data                               | Loop Over Items5                 | Google Sheets1                  |                                                                                                          |
| Google Sheets1                | Google Sheets Read               | Reads lead data                               | Google Sheets                   | Company name is not empty (If)  |                                                                                                          |
| Company name is not empty (If) | If                             | Checks if company name exists and website empty | Google Sheets1                  | Loop Over Items                 | At this stage, the presence of the company name in the lead's profile is verified, and a website search is performed for each company |
| Loop Over Items               | Split In Batches                 | Processes leads with company names            | Company name is not empty (If)   | Google Sheets2, HDW Get Company Website |                                                                                                          |
| HDW Get Company Website       | HDW LinkedIn Node (Google Search) | Searches Google for company website           | Loop Over Items                 | Google Sheets3                 | Get company website                                                                                       |
| Google Sheets2                | Google Sheets Update             | Updates lead row with website URL             | Loop Over Items                 | Google Sheets3, Google Sheets4, Google Sheets6, Google Sheets8 |                                                                                                          |
| Google Sheets3                | Google Sheets Read               | Reads updated lead data                        | Google Sheets2                  | Website is not empty (If)       |                                                                                                          |
| Website is not empty (If)      | If                             | Checks if website exists and product summary empty | Google Sheets3                  | Loop Over Items1                |                                                                                                          |
| Loop Over Items1              | Split In Batches                 | Processes leads with websites                  | Website is not empty (If)        | HDW Site-map, HDW Parser        |                                                                                                          |
| HDW Site-map                 | HDW Web Parser Tool             | Retrieves sitemap from company website        | Loop Over Items1                | HDW Parser                    |                                                                                                          |
| HDW Parser                   | HDW Web Parser Tool             | Scrapes content from sitemap URLs              | HDW Site-map                   | Summarise company website       |                                                                                                          |
| Summarise company website     | LangChain Agent                 | Summarizes company website content             | HDW Parser                    | Google Sheets10                |                                                                                                          |
| Google Sheets10              | Google Sheets Update             | Saves summarized product info                   | Summarise company website       | Google Sheets10                |                                                                                                          |
| HDW Get User Posts           | HDW LinkedIn Node              | Retrieves lead's LinkedIn posts                 | Loop Over Items2                | Aggregate                     | Research lead LN post                                                                                      |
| Aggregate                   | Aggregate                      | Aggregates lead posts text                       | HDW Get User Posts             | Summarise user posts           |                                                                                                          |
| Summarise user posts         | OpenAI Node                   | Summarizes lead posts                            | Aggregate                     | Google Sheets5                |                                                                                                          |
| Google Sheets5              | Google Sheets Update             | Saves posts summary                              | Summarise user posts           | Post summary is empty (If)     |                                                                                                          |
| Post summary is empty (If)    | If                             | Checks if posts summary is missing               | Google Sheets5                | Loop Over Items2               |                                                                                                          |
| Loop Over Items2             | Split In Batches                 | Processes leads missing posts summary            | Post summary is empty (If)      | HDW Get User Posts            |                                                                                                          |
| HDW Get Company News         | HDW LinkedIn Node (Google Search) | Searches Google for company news                 | Loop Over Items3                | Aggregate1                   | Research company News                                                                                      |
| Aggregate1                  | Aggregate                      | Aggregates company news descriptions             | HDW Get Company News           | Summarise company news        |                                                                                                          |
| Summarise company news       | OpenAI Node                   | Summarizes company news                           | Aggregate1                   | Google Sheets7               |                                                                                                          |
| Google Sheets7              | Google Sheets Update             | Saves company news summary                        | Summarise company news        | Company news is empty (If)    |                                                                                                          |
| Company news is empty (If)    | If                             | Checks if company news summary is missing         | Google Sheets7               | Loop Over Items3              |                                                                                                          |
| Loop Over Items3             | Split In Batches                 | Processes leads missing company news summary      | Company news is empty (If)     | HDW Get Company News          |                                                                                                          |
| HDW Get Company Posts        | HDW LinkedIn Node              | Retrieves company LinkedIn posts                   | Loop Over Items4               | Aggregate2                   | Research company LN post                                                                                   |
| Aggregate2                  | Aggregate                      | Aggregates company posts text                      | HDW Get Company Posts          | Summarise company posts       |                                                                                                          |
| Summarise company posts      | OpenAI Node                   | Summarizes company posts                           | Aggregate2                   | Google Sheets9               |                                                                                                          |
| Google Sheets9              | Google Sheets Update             | Saves company posts summary                        | Summarise company posts       | Company post is empty (If)    |                                                                                                          |
| Company post is empty (If)    | If                             | Checks if company post summary is missing           | Google Sheets9               | Loop Over Items4              |                                                                                                          |
| Loop Over Items4             | Split In Batches                 | Processes leads missing company post summary        | Company post is empty (If)     | HDW Get Company Posts         |                                                                                                          |
| Lead Score is empty (If)      | If                             | Checks if lead score is missing                     | Google Sheets11              | Loop Over Items6              |                                                                                                          |
| Loop Over Items6             | Split In Batches                 | Processes leads missing scores                       | Lead Score is empty (If)       | Company Score Analysis       |                                                                                                          |
| Company Score Analysis       | OpenAI Node                   | Scores leads 1-10 based on enriched data             | Loop Over Items6              | Google Sheets12              | You can also change the scoring criteria to assess the probability of need based on your product or business by adjusting the prompt in this node |
| Google Sheets12             | Google Sheets Update             | Saves lead score                                    | Company Score Analysis        | Loop Over Items6              |                                                                                                          |
| Google Sheets13             | Google Sheets Read               | Reads scored leads                                  | Google Sheets12              | Sort                        |                                                                                                          |
| Sort                       | Sort                          | Sorts leads descending by score                      | Google Sheets13              | If2                         |                                                                                                          |
| If2                        | If                             | Filters leads without contact requests               | Sort                        | Limit                       |                                                                                                          |
| Limit                      | Limit                         | Limits connection requests to max 20 per run         | If2                         | Loop Over Items7             | max 200 per week                                                                                          |
| Loop Over Items7            | Split In Batches                 | Processes leads for connection requests              | Limit                       | HDW Send LinkedIn Connection |                                                                                                          |
| HDW Send LinkedIn Connection | HDW LinkedIn Management Node   | Sends connection requests                             | Loop Over Items7             | Google Sheets14             |                                                                                                          |
| Google Sheets14             | Google Sheets Update             | Marks contact request sent                            | HDW Send LinkedIn Connection |                             |                                                                                                          |
| Schedule Trigger            | Schedule Trigger               | Triggers connection request workflow daily at 7 AM   | -                           | Google Sheets13             | Change the schedule for when connection requests will be sent.                                           |
| HDW Get LinkedIn Profile Connections | HDW LinkedIn Management Node   | Retrieves accepted connections                        | Schedule Trigger1            | Split LinkedIn connections to items |                                                                                                          |
| Split LinkedIn connections to items | Code Node                     | Converts connections array to individual items         | HDW Get LinkedIn Profile Connections | Google Sheets15             |                                                                                                          |
| Google Sheets15             | Google Sheets Update             | Marks leads as connected                              | Split LinkedIn connections to items | Google Sheets16             |                                                                                                          |
| Google Sheets16             | Google Sheets Read               | Reads connected leads for messaging                    | Google Sheets15             | Loop Over Items8             |                                                                                                          |
| Loop Over Items8            | Split In Batches                 | Processes connected leads for messaging                | Google Sheets16             | HDW LinkedIn Send Message   |                                                                                                          |
| HDW LinkedIn Send Message   | HDW LinkedIn Management Node   | Sends personalized messages                            | Loop Over Items8             | Google Sheets17             | In this node, you need to modify the message that will be automatically sent to the user after the connection is confirmed. |
| Google Sheets17             | Google Sheets Update             | Marks message sent                                    | HDW LinkedIn Send Message   |                             |                                                                                                          |
| Schedule Trigger1           | Schedule Trigger               | Triggers messaging workflow daily at 7 AM             | -                           | HDW Get LinkedIn Profile Connections | Change the schedule for when connection request responses will be checked and messages will be sent.      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Chat Trigger Node**  
   - Type: LangChain Chat Trigger  
   - Purpose: Receive ICP description input from user chat  
   - No special parameters needed

2. **Add OpenAI Chat Model Node**  
   - Type: LangChain OpenAI Chat Model  
   - Model: gpt-4o  
   - Connect input from Chat Trigger  
   - Configure OpenAI API credentials

3. **Add Simple Memory Node**  
   - Type: LangChain Memory Buffer Window  
   - Connect input from OpenAI Chat Model  
   - Purpose: Maintain chat context

4. **Add AI Agent Node for ICP Translation**  
   - Type: LangChain Agent  
   - System prompt: Detailed instructions to convert ICP text into LinkedIn Sales Navigator filters with industry list  
   - Connect input from Simple Memory and OpenAI Chat Model  
   - Enable output parser

5. **Add Structured Output Parser Node**  
   - Type: LangChain Structured Output Parser  
   - JSON schema example for Sales Navigator parameters  
   - Connect input from AI Agent

6. **Add Split Out Node**  
   - Type: Split Out  
   - Connect input from Structured Output Parser  
   - Purpose: Split array of filters into individual items

7. **Add HDW LinkedIn Search Node**  
   - Type: HDW LinkedIn Community Node  
   - Configure to use dynamic parameters from Split Out node (keywords, titles, companies, location, industry, company sizes)  
   - Set resource to "search"  
   - Connect input from Split Out  
   - Configure HDW API credentials

8. **Add Split In Batches Node (Loop Over Items5)**  
   - Batch size: default or tuned for performance  
   - Connect input from HDW LinkedIn Search

9. **Add Google Sheets Append or Update Node**  
   - Configure to append or update lead data in Google Sheet  
   - Map columns: Name, URN, URL, Headline, Location, Company, Industry, etc.  
   - Connect input from Loop Over Items5  
   - Configure Google Sheets OAuth2 credentials

10. **Add Google Sheets Read Node (Google Sheets1)**  
    - Reads lead data for enrichment  
    - Connect input from Google Sheets

11. **Add If Node (Company name is not empty)**  
    - Condition: Current company is not empty AND Website is empty  
    - Connect input from Google Sheets1

12. **Add Split In Batches Node (Loop Over Items)**  
    - Processes leads with company names for website enrichment  
    - Connect input from If node (true branch)

13. **Add HDW LinkedIn Node (HDW Get Company Website)**  
    - Resource: Google Search  
    - Query: "{Current company} company Website"  
    - Connect input from Loop Over Items  
    - Configure HDW API credentials

14. **Add Google Sheets Update Node (Google Sheets2)**  
    - Updates lead row with website URL  
    - Connect input from Loop Over Items (true branch) and HDW Get Company Website

15. **Add Google Sheets Read Node (Google Sheets3)**  
    - Reads updated lead data  
    - Connect input from Google Sheets2

16. **Add If Node (Website is not empty)**  
    - Condition: Website is not empty AND Product Summary is empty  
    - Connect input from Google Sheets3

17. **Add Split In Batches Node (Loop Over Items1)**  
    - Processes leads with websites for scraping  
    - Connect input from If node (true branch)

18. **Add HDW Web Parser Tool Nodes (HDW Site-map and HDW Parser)**  
    - HDW Site-map: Gets sitemap from Website URL  
    - HDW Parser: Scrapes content from sitemap URLs  
    - Connect HDW Site-map output to HDW Parser input  
    - Connect Loop Over Items1 to HDW Site-map

19. **Add LangChain Agent Node (Summarise company website)**  
    - Summarizes scraped website content  
    - Connect input from HDW Parser

20. **Add Google Sheets Update Node (Google Sheets10)**  
    - Saves product summary to sheet  
    - Connect input from Summarise company website

21. **Add HDW LinkedIn Node (HDW Get User Posts)**  
    - Retrieves lead's LinkedIn posts  
    - Connect input from Loop Over Items2 (to be created)  
    - Configure HDW API credentials

22. **Add Aggregate Node**  
    - Aggregates lead posts text and reposts  
    - Connect input from HDW Get User Posts

23. **Add OpenAI Node (Summarise user posts)**  
    - Summarizes aggregated posts  
    - Connect input from Aggregate

24. **Add Google Sheets Update Node (Google Sheets5)**  
    - Saves posts summary  
    - Connect input from Summarise user posts

25. **Add If Node (Post summary is empty)**  
    - Checks if posts summary is missing  
    - Connect input from Google Sheets5

26. **Add Split In Batches Node (Loop Over Items2)**  
    - Processes leads missing posts summary  
    - Connect input from If node (true branch)

27. **Add HDW LinkedIn Node (HDW Get Company News)**  
    - Google Search for company news  
    - Connect input from Loop Over Items3 (to be created)

28. **Add Aggregate Node (Aggregate1)**  
    - Aggregates news descriptions  
    - Connect input from HDW Get Company News

29. **Add OpenAI Node (Summarise company news)**  
    - Summarizes company news  
    - Connect input from Aggregate1

30. **Add Google Sheets Update Node (Google Sheets7)**  
    - Saves company news summary  
    - Connect input from Summarise company news

31. **Add If Node (Company news is empty)**  
    - Checks if company news summary is missing  
    - Connect input from Google Sheets7

32. **Add Split In Batches Node (Loop Over Items3)**  
    - Processes leads missing company news summary  
    - Connect input from If node (true branch)

33. **Add HDW LinkedIn Node (HDW Get Company Posts)**  
    - Retrieves company LinkedIn posts  
    - Connect input from Loop Over Items4 (to be created)

34. **Add Aggregate Node (Aggregate2)**  
    - Aggregates company posts text  
    - Connect input from HDW Get Company Posts

35. **Add OpenAI Node (Summarise company posts)**  
    - Summarizes company posts  
    - Connect input from Aggregate2

36. **Add Google Sheets Update Node (Google Sheets9)**  
    - Saves company posts summary  
    - Connect input from Summarise company posts

37. **Add If Node (Company post is empty)**  
    - Checks if company post summary is missing  
    - Connect input from Google Sheets9

38. **Add Split In Batches Node (Loop Over Items4)**  
    - Processes leads missing company post summary  
    - Connect input from If node (true branch)

39. **Add If Node (Lead Score is empty)**  
    - Checks if lead score is missing  
    - Connect input from Google Sheets11 (to be created)

40. **Add Split In Batches Node (Loop Over Items6)**  
    - Processes leads missing scores  
    - Connect input from If node (true branch)

41. **Add OpenAI Node (Company Score Analysis)**  
    - Scores leads 1-10 based on enriched data  
    - System prompt defines scoring criteria  
    - Connect input from Loop Over Items6

42. **Add Google Sheets Update Node (Google Sheets12)**  
    - Saves lead score  
    - Connect input from Company Score Analysis

43. **Add Google Sheets Read Node (Google Sheets13)**  
    - Reads scored leads  
    - Connect input from Google Sheets12

44. **Add Sort Node**  
    - Sorts leads descending by Lead Score  
    - Connect input from Google Sheets13

45. **Add If Node (If2)**  
    - Filters leads without contact requests  
    - Connect input from Sort

46. **Add Limit Node**  
    - Limits connection requests to max 20 per run  
    - Connect input from If2

47. **Add Split In Batches Node (Loop Over Items7)**  
    - Processes leads for connection requests  
    - Connect input from Limit

48. **Add HDW LinkedIn Management Node (HDW Send LinkedIn Connection)**  
    - Sends connection requests  
    - Connect input from Loop Over Items7  
    - Configure HDW API credentials

49. **Add Google Sheets Update Node (Google Sheets14)**  
    - Marks contact request sent  
    - Connect input from HDW Send LinkedIn Connection

50. **Add Schedule Trigger Node**  
    - Triggers daily at 7 AM to send connection requests  
    - Connect output to Google Sheets13

51. **Add HDW LinkedIn Management Node (HDW Get LinkedIn Profile Connections)**  
    - Retrieves accepted connections  
    - Connect input from Schedule Trigger1 (to be created)  
    - Configure HDW API credentials

52. **Add Code Node (Split LinkedIn connections to items)**  
    - Converts connections array to individual items  
    - Connect input from HDW Get LinkedIn Profile Connections

53. **Add Google Sheets Update Node (Google Sheets15)**  
    - Marks leads as connected  
    - Connect input from Code Node

54. **Add Google Sheets Read Node (Google Sheets16)**  
    - Reads connected leads for messaging  
    - Connect input from Google Sheets15

55. **Add Split In Batches Node (Loop Over Items8)**  
    - Processes connected leads for messaging  
    - Connect input from Google Sheets16

56. **Add HDW LinkedIn Management Node (HDW LinkedIn Send Message)**  
    - Sends personalized messages  
    - Connect input from Loop Over Items8  
    - Configure HDW API credentials  
    - Customize message text

57. **Add Google Sheets Update Node (Google Sheets17)**  
    - Marks message sent  
    - Connect input from HDW LinkedIn Send Message

58. **Add Schedule Trigger1 Node**  
    - Triggers daily at 7 AM to check connections and send messages  
    - Connect output to HDW Get LinkedIn Profile Connections

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                          |
|--------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| This workflow uses the HDW LinkedIn community node, which is only available on self-hosted n8n instances. It will not work on n8n.cloud.    | https://www.npmjs.com/package/n8n-nodes-hdw                                                            |
| HDW API key is required and can be obtained at https://app.horizondatawave.ai                                                              |                                                                                                        |
| Google Sheets must be set up with columns: Name, URN, URL, Headline, Location, Current company, Industry, etc.                             | Google Sheets template: https://docs.google.com/spreadsheets/d/18w5jT_Gx20bhW_UfU8IiUkEO0xMioD7XZZC2sShQItA/edit?usp=sharing |
| OpenAI API key is required for GPT-4o usage                                                                                               |                                                                                                        |
| LinkedIn connection request limits are approximately 100-200 per week; the workflow includes safeguards to avoid exceeding these limits.   |                                                                                                        |
| Customize AI prompts in "AI Agent: ICP -> LinkedIn search filters" and "Company Score Analysis" nodes to tailor to your product/service.   |                                                                                                        |
| Customize message templates in "HDW LinkedIn Send Message" node for personalized outreach.                                                 |                                                                                                        |
| Schedule triggers run daily at 7 AM by default; adjust timing as needed.                                                                   |                                                                                                        |
| Always use automation tools responsibly and comply with LinkedIn's terms of service.                                                       |                                                                                                        |

---

This document provides a comprehensive, structured reference for understanding, reproducing, and modifying the Automated LinkedIn Lead Generation, Scoring & Communication workflow using n8n and HDW LinkedIn nodes.