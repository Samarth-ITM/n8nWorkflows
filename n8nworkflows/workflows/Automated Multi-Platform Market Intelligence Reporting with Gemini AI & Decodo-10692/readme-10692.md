Automated Multi-Platform Market Intelligence Reporting with Gemini AI & Decodo

https://n8nworkflows.xyz/workflows/automated-multi-platform-market-intelligence-reporting-with-gemini-ai---decodo-10692


# Automated Multi-Platform Market Intelligence Reporting with Gemini AI & Decodo

### 1. Workflow Overview

This workflow automates multi-platform market intelligence reporting using Decodo web scraping APIs combined with advanced AI analysis via Google Gemini models. It targets marketing and competitive intelligence teams seeking structured insights on public discussion, competitor activity, and trends across Facebook, Instagram, and general Google search results.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Task Initialization**: Daily trigger sets static parameters (topic, regions, search terms) and builds a granular task list for each platform-region combination.
- **1.2 Task Processing Loop**: Iterates over each platform-region task, routing tasks to platform-specific scraping and analysis chains.
- **1.3 Platform-Specific Data Gathering & Cleaning**: For each platform (Facebook, Instagram, Google Search), the workflow:
  - Finds relevant URLs via Decodo-powered Google site-limited search
  - Scrapes full HTML content of top results
  - Cleans HTML to extract meaningful plain text
- **1.4 AI-Powered Text Analysis**: Uses specialized Google Gemini language models to analyze cleaned text per platform, focusing on sentiment, competitor activity, and trends. Outputs structured JSON summaries.
- **1.5 Combining Analyses & Final Report Generation**: Merges platform JSON outputs, then passes combined data to an AI agent that drafts a professional executive email report.
- **1.6 Email Delivery**: Sends the final daily intelligence email to designated recipients using Gmail OAuth2.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Task Initialization

- **Overview**: This block triggers the workflow at 6:00 AM daily, defines static search parameters, and builds a detailed list of platform-region search tasks.
- **Nodes Involved**: Schedule Trigger, Set - Regions, Build Task List, Loop: Each Task
- **Node Details**:
  - **Schedule Trigger**
    - Type: Trigger node
    - Configuration: Fires daily at 6:00 AM
    - Inputs: None (initial entry)
    - Outputs: Starts execution flow to Set - Regions
    - Failure Modes: Scheduling misconfiguration, time zone issues
  - **Set - Regions**
    - Type: Set node
    - Configuration: Holds static JSON with keys: topic, regions array, search_terms array
    - Key Variables: `topic`, `regions`, `search_terms`
    - Inputs: From Schedule Trigger
    - Outputs: To Build Task List
  - **Build Task List**
    - Type: Code node (JavaScript)
    - Configuration: Constructs tasks for each combination of region and platform ([facebook.com, instagram.com, google.com]) with a search query including topic and a date limit (1 month ago)
    - Expressions: Calculates `afterDate` (ISO date string one month prior)
    - Inputs: From Set - Regions
    - Outputs: Array of tasks, each having topic, region, platform, search_terms, afterDate, query
  - **Loop: Each Task**
    - Type: SplitInBatches node
    - Configuration: Processes one task at a time from the task list
    - Inputs: From Build Task List
    - Outputs: To Route by Platform and Draft Executive Email (final merge loop)
- **Edge Cases**: Empty regions/topic input, date calculations errors, large task list causing performance issues

---

#### 2.2 Task Routing by Platform

- **Overview**: Directs each platform-region task to the appropriate platform-specific scraping and analysis chain.
- **Nodes Involved**: Route by Platform
- **Node Details**:
  - **Route by Platform**
    - Type: Switch node
    - Configuration: Routes tasks based on `platform` field value (`facebook.com`, `instagram.com`, `google.com`)
    - Inputs: From Loop: Each Task
    - Outputs: To FB: Find URLs, IG: Find URLs, Web: Find URLs respectively
- **Edge Cases**: Unexpected platform values, case-sensitivity issues, missing platform field

---

#### 2.3 Facebook Data Gathering & Analysis Chain

- **Overview**: Finds Facebook posts via Google site-limited search, scrapes post HTML, cleans it, and analyzes text for market insights.
- **Nodes Involved**: FB: Find URLs, FB: Scrape Post, FB: Clean HTML, FB: Analyze Text
- **Node Details**:
  - **FB: Find URLs**
    - Type: Decodo node (Google Search operation)
    - Configuration: Performs Google search with query including topic, region, platform (facebook.com), and date filter
    - Inputs: From Route by Platform
    - Credentials: Decodo API token required
    - Outputs: JSON with search results containing URLs
    - Retry: Enabled on failure
    - Failure Modes: API rate limits, invalid credentials, empty search results
  - **FB: Scrape Post**
    - Type: Decodo node (Web scraping)
    - Configuration: Scrapes full HTML content of the top organic search result URL
    - Inputs: From FB: Find URLs
    - Credentials: Decodo API
    - Retry: Enabled
    - Outputs: Raw HTML content
  - **FB: Clean HTML**
    - Type: Code node (JavaScript)
    - Configuration: Removes scripts, styles, noscript tags, strips HTML tags, decodes entities, collapses whitespace, truncates if >10k chars
    - Inputs: Raw HTML from FB: Scrape Post
    - Outputs: Clean plain text (`text_clean`)
    - Edge Cases: Unexpected HTML structure, empty input, script-heavy pages
  - **FB: Analyze Text**
    - Type: Langchain Agent node with Google Gemini Chat Model 2
    - Configuration: Processes cleaned text with a prompt tailored for Facebook market intelligence analysis, outputs structured JSON summarizing public discussion, sentiment, competitor activity, trends
    - Inputs: Clean text from FB: Clean HTML
    - Outputs: JSON analysis
    - Edge Cases: AI prompt failures, invalid JSON output, API quota exceeded

---

#### 2.4 Instagram Data Gathering & Analysis Chain

- **Overview**: Similar to Facebook chain but focused on Instagram posts and influencer trends.
- **Nodes Involved**: IG: Find URLs, IG: Scrape Post, IG: Clean HTML, IG: Analyze Text
- **Node Details**:
  - **IG: Find URLs**
    - Type: Decodo node (Google Search)
    - Configuration: Google search limited to `site:instagram.com` with date filter
    - Inputs: From Route by Platform
    - Credentials: Decodo API
    - Retry: Enabled
  - **IG: Scrape Post**
    - Type: Decodo node (Web scraping)
    - Inputs: From IG: Find URLs
    - Outputs: Raw HTML content of top Instagram post URL
  - **IG: Clean HTML**
    - Type: Code node
    - Same cleaning logic as Facebook
  - **IG: Analyze Text**
    - Type: Langchain Agent with Google Gemini Chat Model 3
    - Configuration: Prompt tailored to Instagram data analyzing audience opinion, sentiment, promotions, influencer collaborations
    - Outputs: Platform-specific structured JSON
- **Edge Cases**: Instagram page dynamic content, empty search results, API limits

---

#### 2.5 General Web (Google Search) Data Gathering & Analysis Chain

- **Overview**: Scrapes general Google search results (news, blogs), cleans HTML, and analyzes for media topics, public interest, and marketing patterns.
- **Nodes Involved**: Web: Find URLs, Web: Scrape Page, Web: Clean HTML, Web: Analyze Text
- **Node Details**:
  - **Web: Find URLs**
    - Decodo node searching Google with topic and date filters (no site restriction)
    - Inputs: From Route by Platform
  - **Web: Scrape Page**
    - Decodo node scraping top organic result HTML
  - **Web: Clean HTML**
    - Code node with same cleaning logic as others
  - **Web: Analyze Text**
    - Langchain Agent with Google Gemini Chat Model 4
    - Prompt focused on competitive intelligence and marketing insights from general web content
- **Edge Cases**: Broad search results, noisy data, inconsistent HTML, AI model errors

---

#### 2.6 Combining Platform Analyses and Final Executive Email Generation

- **Overview**: Merges the JSON outputs from Facebook, Instagram, and Web analyses, then drafts a consolidated executive email summarizing all findings.
- **Nodes Involved**: Combine All Analyses, Draft Executive Email, Send Daily Report
- **Node Details**:
  - **Combine All Analyses**
    - Type: Merge node
    - Configuration: Merges 3 inputs (Facebook, Instagram, Web analyses) into one dataset
    - Inputs: From FB: Analyze Text, IG: Analyze Text, Web: Analyze Text
    - Outputs: To Draft Executive Email
  - **Draft Executive Email**
    - Type: Langchain Agent with Google Gemini Chat Model (general)
    - Configuration: Uses a detailed prompt to create a professional email report with two main sections: People's Opinions and Competitors & Promotions, dynamically including all regions and platforms
    - Inputs: Combined JSON from Merge node
    - Outputs: Plain text email body
  - **Send Daily Report**
    - Type: Gmail node
    - Configuration: Sends the email to specified recipients using Gmail OAuth2 credentials
    - Inputs: Email text from Draft Executive Email
    - Parameters: Subject includes topic, customizable recipient email
- **Edge Cases**: Email send failures, incomplete data merges, AI output formatting issues

---

### 3. Summary Table

| Node Name           | Node Type                          | Functional Role                               | Input Node(s)                    | Output Node(s)                | Sticky Note                                                                                                                    |
|---------------------|----------------------------------|----------------------------------------------|---------------------------------|------------------------------|-------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger     | Trigger                          | Starts workflow daily at 6:00 AM              | None                            | Set - Regions                 | Starts the entire workflow automatically at a set time. This node is set to run daily at 6:00 AM.                              |
| Set - Regions        | Set                              | Defines static topic, regions, search terms  | Schedule Trigger                | Build Task List               | Defines the static list of regions and search terms. This is the master list that the workflow will use for its searches.        |
| Build Task List      | Code                             | Builds platform-region tasks with date filter| Set - Regions                  | Loop: Each Task               | Multiplies the tasks by creating a separate job for each platform and each region. This node builds the main list of tasks.     |
| Loop: Each Task      | SplitInBatches                   | Processes tasks one by one                     | Build Task List, Combine All Analyses | Route by Platform, Draft Executive Email | Processes each job (e.g., "Facebook in Germany") one by one. This ensures every platform in every region is scraped and analyzed. |
| Route by Platform    | Switch                          | Routes tasks to platform-specific chains      | Loop: Each Task                | FB: Find URLs, IG: Find URLs, Web: Find URLs | Directs each job to the correct processing chain. It sends Facebook jobs to the Facebook group, Instagram jobs to Instagram, etc. |
| FB: Find URLs        | Decodo (Google Search)           | Finds Facebook URLs via Google site search    | Route by Platform              | FB: Scrape Post              | Searches Google specifically on site:facebook.com for relevant posts. This node finds the target URLs to scrape.               |
| FB: Scrape Post      | Decodo (Web Scrape)              | Scrapes full HTML of Facebook posts           | FB: Find URLs                  | FB: Clean HTML               | Scrapes the full HTML content from the Facebook URL found in the search. This gets the raw, messy page data.                   |
| FB: Clean HTML       | Code                            | Cleans HTML, strips tags, decodes entities    | FB: Scrape Post                | FB: Analyze Text             | Strips all HTML tags, scripts, and styles from the scraped data. This converts the raw page into clean, analyzable text.       |
| FB: Analyze Text     | Langchain Agent (Google Gemini) | Analyzes Facebook text for insights, sentiment| FB: Clean HTML                 | Combine All Analyses          | Analyzes the clean text specifically for Facebook insights. It structures findings on public discussion and competitor activity into JSON. |
| IG: Find URLs        | Decodo (Google Search)           | Finds Instagram URLs via Google site search   | Route by Platform              | IG: Scrape Post              |                                                                                                                                 |
| IG: Scrape Post      | Decodo (Web Scrape)              | Scrapes full HTML of Instagram content        | IG: Find URLs                  | IG: Clean HTML               |                                                                                                                                 |
| IG: Clean HTML       | Code                            | Cleans Instagram HTML                          | IG: Scrape Post                | IG: Analyze Text             |                                                                                                                                 |
| IG: Analyze Text     | Langchain Agent (Google Gemini) | Analyzes Instagram text for marketing trends  | IG: Clean HTML                 | Combine All Analyses          |                                                                                                                                 |
| Web: Find URLs       | Decodo (Google Search)           | Finds general web URLs via Google search      | Route by Platform              | Web: Scrape Page             |                                                                                                                                 |
| Web: Scrape Page     | Decodo (Web Scrape)              | Scrapes full HTML of web pages                 | Web: Find URLs                 | Web: Clean HTML              |                                                                                                                                 |
| Web: Clean HTML      | Code                            | Cleans web HTML                                | Web: Scrape Page               | Web: Analyze Text            |                                                                                                                                 |
| Web: Analyze Text    | Langchain Agent (Google Gemini) | Analyzes web text for media, marketing insights| Web: Clean HTML                | Combine All Analyses          |                                                                                                                                 |
| Combine All Analyses | Merge                           | Merges Facebook, Instagram, and Web analyses | FB: Analyze Text, IG: Analyze Text, Web: Analyze Text | Loop: Each Task           | Collects all the individual JSON analyses from all three platforms. This node waits until all scraping is finished before passing the combined data to the final AI. |
| Draft Executive Email| Langchain Agent (Google Gemini) | Combines all JSON inputs into a final email   | Combine All Analyses           | Send Daily Report            | Receives the combined JSON from all platforms and regions. Its job is to write the final, human-readable email summary for leadership. |
| Send Daily Report    | Gmail                           | Sends final email report                        | Draft Executive Email          | None                        | Sends the final email generated by the AI agent to the specified recipients.                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a "Schedule Trigger" node**
   - Type: Schedule Trigger
   - Set to run daily at 6:00 AM
   - Connect output to next node

2. **Create a "Set" node named "Set - Regions"**
   - Type: Set
   - Add JSON mode with keys:
     - `topic`: your target market research topic (string)
     - `regions`: array of region names, e.g., ["Region_1", "Region_2", "Region_3"]
     - `search_terms`: array of search keywords relevant to your topic
   - Connect input from Schedule Trigger

3. **Create a "Code" node named "Build Task List"**
   - Type: Code (JavaScript)
   - Paste code that:
     - Reads topic, regions, search_terms from input JSON
     - Defines platforms array: ["facebook.com", "instagram.com", "google.com"]
     - Calculates `afterDate` as ISO string for one month ago
     - Outputs one task per platform-region with search query including topic, platform, and afterDate filter
   - Connect input from Set - Regions

4. **Create a "SplitInBatches" node named "Loop: Each Task"**
   - Type: SplitInBatches
   - No special configuration (default batch size 1)
   - Connect input from Build Task List

5. **Create a "Switch" node named "Route by Platform"**
   - Type: Switch
   - Add three rules to match `$json.platform` exactly to "facebook.com", "instagram.com", "google.com"
   - Connect input from Loop: Each Task

6. **Build Facebook scraping chain:**

   - **Decodo node "FB: Find URLs"**
     - Operation: Google Search
     - Query: Use expression with current task's query + `site:facebook.com`
     - Geo: Current region from task
     - Credentials: Decodo API
     - Connect input from Route by Platform for Facebook output

   - **Decodo node "FB: Scrape Post"**
     - URL: Extract top result URL from previous node's JSON
     - Credentials: Decodo API
     - Connect input from FB: Find URLs

   - **Code node "FB: Clean HTML"**
     - JavaScript to remove scripts, styles, tags, decode entities, truncate long text
     - Connect input from FB: Scrape Post

   - **Langchain Agent node "FB: Analyze Text"**
     - Model: Google Gemini Chat Model 2
     - Prompt: Custom prompt targeting Facebook data analysis (as per JSON)
     - Input text: expression extracting `text_clean` from previous node
     - Connect input from FB: Clean HTML

7. **Build Instagram scraping chain:** (similar to Facebook, replacing site with instagram.com and adjusting prompt)

   - Decodo "IG: Find URLs"
   - Decodo "IG: Scrape Post"
   - Code "IG: Clean HTML"
   - Langchain Agent "IG: Analyze Text" with Google Gemini Chat Model 3

8. **Build Web scraping chain:** (for general Google search)

   - Decodo "Web: Find URLs" (without site restriction)
   - Decodo "Web: Scrape Page"
   - Code "Web: Clean HTML"
   - Langchain Agent "Web: Analyze Text" with Google Gemini Chat Model 4

9. **Create "Merge" node "Combine All Analyses"**
   - Number of inputs: 3 (Facebook, Instagram, Web)
   - Connect inputs from FB: Analyze Text, IG: Analyze Text, Web: Analyze Text

10. **Connect output of "Combine All Analyses" back to Loop: Each Task** (to allow batch loop continuation)

11. **Create Langchain Agent node "Draft Executive Email"**
    - Model: Google Gemini Chat Model (default)
    - Prompt: Detailed prompt to combine multiple platform JSONs into a professional daily executive email
    - Input text: expression reading merged analyses output
    - Connect input from Loop: Each Task (second output)

12. **Create Gmail node "Send Daily Report"**
    - Credential: Gmail OAuth2 with your account
    - To: Your intended recipient email
    - Subject: Include your topic dynamically
    - Message: Use output of Draft Executive Email
    - Connect input from Draft Executive Email

13. **Ensure all nodes have correct credential references (Decodo API, Gmail OAuth2)**

14. **Test workflow with sample data by replacing placeholders `[YOUR_TOPIC_HERE]`, `[Region_1]`, `[Search_Term_1]`, and emails**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------|
| How to Set Up **Decodo** Credentials in n8n: 1) Activate an Advanced Web Scraping API plan on Decodo dashboard (free trial available). 2) Locate and copy your authentication token from Decodo Web Scraping API page. 3) In n8n, create new 'Decodo Credentials API' credential and paste your token. For detailed instructions, visit: github.com/Decodo/n8n-nodes-decodo/tree/main                                                                                                         | Decodo Credentials Setup (Sticky Note 15)                                                             |
| Facebook Group chain searches Facebook via Google site search, scrapes posts, cleans HTML, and analyzes for insights on public discussion and competitors.                                                                                                                                                                                                                                                                                                                               | Sticky Note                                                                                           |
| Instagram Group chain finds Instagram posts similarly, analyzes trends and promotions.                                                                                                                                                                                                                                                                                                                                                                                                     | Sticky Note                                                                                           |
| Google Search Group chain scrapes general web results (news, blogs), cleans data, and summarizes topics and marketing patterns.                                                                                                                                                                                                                                                                                                                                                          | Sticky Note                                                                                           |
| Workflow is designed for daily automated execution producing a combined, human-readable market intelligence email report.                                                                                                                                                                                                                                                                                                                                                                | Sticky Note                                                                                           |

---

**Disclaimer:** The text provided is exclusively derived from an automated n8n workflow. It fully respects content policies and contains no illegal, offensive, or protected material. All data handled is legal and public.