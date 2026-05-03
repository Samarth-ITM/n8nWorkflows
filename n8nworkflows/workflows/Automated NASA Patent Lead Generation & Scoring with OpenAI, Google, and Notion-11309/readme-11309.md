Automated NASA Patent Lead Generation & Scoring with OpenAI, Google, and Notion

https://n8nworkflows.xyz/workflows/automated-nasa-patent-lead-generation---scoring-with-openai--google--and-notion-11309


# Automated NASA Patent Lead Generation & Scoring with OpenAI, Google, and Notion

### 1. Workflow Overview

This workflow, titled **Automated NASA Patent Lead Generation & Scoring with OpenAI, Google, and Notion**, is designed to automate the scouting and outreach process for commercial partners interested in NASA patents. Its primary users are innovation managers, tech transfer offices, and business development representatives who want to efficiently identify startups aligned with NASA technologies and automatically generate personalized outreach communications.

The workflow is logically divided into the following blocks:

- **1.1 Data Acquisition:** Initiates the process by receiving a trigger, setting search parameters, and retrieving NASA patent data based on a defined keyword.
  
- **1.2 Company Identification:** For each patent, searches Google to find potential companies/startups related to the patent technology and extracts their website URLs.
  
- **1.3 Company Enrichment:** Crawls the identified company websites to gather detailed content about their business.
  
- **1.4 AI Analysis & Email Drafting:** Uses OpenAI to analyze the fit between the patent and the company and drafts a tailored outreach email.
  
- **1.5 Lead Filtering & Enrichment:** Filters leads by AI-assigned score, enriches high-scoring leads by searching for LinkedIn profiles, and saves the enriched lead data into a Notion database.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Data Acquisition

**Overview:**  
Starts the workflow manually, sets the keyword for patent search, and fetches relevant patent data from NASA’s Tech Transfer API.

**Nodes Involved:**  
- Manual Trigger  
- Configuration  
- NASA Patents API  
- Loop Over Patents

**Node Details:**

- **Manual Trigger**  
  - Type: Trigger node  
  - Role: Starts workflow execution manually (user-initiated)  
  - Config: No parameters, default manual trigger  
  - Input: None  
  - Output: Triggers the next node (Configuration)  
  - Failure modes: None significant; user must trigger manually  
  - Version: 1

- **Configuration**  
  - Type: Set node  
  - Role: Defines workflow parameters, specifically the keyword for NASA patent search  
  - Config: Assigns a string variable `keyword` (default value: "robotics")  
  - Expressions: None dynamic, keyword is static but can be modified  
  - Input: Output of Manual Trigger  
  - Output: Passes `keyword` to NASA Patents API  
  - Failure modes: If keyword is empty or invalid, NASA API may return no results  
  - Version: 3.4

- **NASA Patents API**  
  - Type: HTTP Request  
  - Role: Queries NASA Tech Transfer API to fetch patents matching the keyword  
  - Config: GET request to `https://api.nasa.gov/techtransfer/patent/?engine&query={{ $json.keyword }}&api_key=<NASA API Key>` (API key placeholder to be replaced)  
  - Expressions: URL dynamically built using the `keyword` set in Configuration node  
  - Input: Receives `keyword`  
  - Output: JSON response with patent data array  
  - Failure modes: API key missing or invalid; network errors; empty responses if no patents found  
  - Version: 4.3

- **Loop Over Patents**  
  - Type: SplitInBatches  
  - Role: Processes each patent record individually to enable sequential handling downstream  
  - Config: Default batch size (likely 1) to handle patents one by one  
  - Input: List of patents from NASA Patents API  
  - Output: One patent item per execution cycle to next nodes  
  - Failure modes: Empty input array leads to no iterations; large data sets may increase runtime  
  - Version: 3

---

#### 2.2 Company Identification

**Overview:**  
For each patent, searches Google using Apify’s `google-search-scraper` actor to find potential companies/startups related to the patent technology, then extracts the company name and website URL.

**Nodes Involved:**  
- Google Search - Find Company  
- Extract Company URL

**Node Details:**

- **Google Search - Find Company**  
  - Type: Apify node (Actor run)  
  - Role: Executes Google search to find companies related to patent keywords  
  - Config: Runs an Apify actor, likely `google-search-scraper` configured with patent keywords (implicit from input)  
  - Input: Single patent data item (contains title, etc.)  
  - Output: Search results JSON with organic results  
  - Credentials: Requires Apify API key  
  - Failure modes: Apify actor failure, rate limiting, or empty search results  
  - Version: 1

- **Extract Company URL**  
  - Type: Code node (JavaScript)  
  - Role: Parses Apify search results to extract the top company URL and name  
  - Config: Custom JS code snippet:  
    ```js
    const results = items[0].json.organicResults;
    return results && results.length > 0 ? [{ json: { targetUrl: results[0].url, companyName: results[0].title } }] : [];
    ```  
  - Input: Search results JSON  
  - Output: JSON with `targetUrl` and `companyName` of the top search result  
  - Failure modes: No organic results (returns empty array, stops downstream processing)  
  - Version: 2

---

#### 2.3 Company Enrichment

**Overview:**  
Crawls the extracted company website using an Apify actor to gather detailed content and business information for further AI analysis.

**Nodes Involved:**  
- Crawl Company Website

**Node Details:**

- **Crawl Company Website**  
  - Type: Apify node (Actor run)  
  - Role: Runs a website content crawler actor on the company URL  
  - Config: Runs Apify actor (e.g., `website-content-crawler`) using `targetUrl` from previous node  
  - Input: JSON with company URL and name  
  - Output: Crawled website data (text, metadata, etc.)  
  - Credentials: Requires Apify API key  
  - Failure modes: Website unreachable, crawler timeout, or empty crawl results  
  - Version: 1

---

#### 2.4 AI Analysis & Email Drafting

**Overview:**  
Uses OpenAI via LangChain node to analyze the fit between the NASA patent technology and the identified company, then drafts a personalized outreach email and assigns a score.

**Nodes Involved:**  
- Analyze Fit & Draft Email

**Node Details:**

- **Analyze Fit & Draft Email**  
  - Type: LangChain OpenAI node  
  - Role: Sends company and patent data to OpenAI to generate a fit score and draft outreach email  
  - Config: Operation set to "message" (chat completion)  
  - Input: Combined data from patent and company crawl  
  - Output: JSON including fields like `score`, `email` (recipient), and `draft_email` (message body)  
  - Credentials: Requires OpenAI API key  
  - Failure modes: API quota exceeded, timeout, malformed prompt causing incomplete response  
  - Version: 2

---

#### 2.5 Lead Filtering & Enrichment

**Overview:**  
Filters leads with high AI-assigned scores (matching "S" or "A" regex), enriches them by searching Google for LinkedIn profiles, and saves qualified leads with full details in a Notion database.

**Nodes Involved:**  
- High Score Filter  
- Google Search - Find LinkedIn  
- Create Notion Lead

**Node Details:**

- **High Score Filter**  
  - Type: If node  
  - Role: Passes only leads whose `score` matches regex pattern `S|A` (interpreted as high scores)  
  - Config: Condition on `$json.score` with regex `"S|A"`  
  - Input: Output from AI analysis node  
  - Output: True branch leads to LinkedIn search; false branch discards lead  
  - Failure modes: Missing or malformed `score` field causes condition to fail  
  - Version: 2.2

- **Google Search - Find LinkedIn**  
  - Type: Apify node (Actor run)  
  - Role: Runs Google search actor to find LinkedIn profiles of the company or contact  
  - Config: Apify actor similar to previous Google search, tailored for LinkedIn discovery  
  - Input: Qualified leads from filter node  
  - Output: LinkedIn profile URL in organicResults  
  - Credentials: Requires Apify API key  
  - Failure modes: No LinkedIn profile found, actor failure  
  - Version: 1

- **Create Notion Lead**  
  - Type: Notion node  
  - Role: Creates a new page in a Notion database with enriched lead details  
  - Config:  
    - Database ID placeholder to be replaced by user  
    - Maps properties: Company (text), Website (URL), LinkedIn (URL), Email, Score (number), Draft Email (text), NASA Tech (text from patent title)  
  - Input: Aggregated data including LinkedIn URL, AI email draft, patent title  
  - Output: Confirmation of page creation  
  - Credentials: Requires Notion OAuth2 credentials with write access  
  - Failure modes: Incorrect database ID, permission errors, missing required fields  
  - Version: 2.2

---

### 3. Summary Table

| Node Name                 | Node Type                     | Functional Role                                  | Input Node(s)             | Output Node(s)           | Sticky Note                                                                                                                  |
|---------------------------|-------------------------------|-------------------------------------------------|---------------------------|--------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Manual Trigger            | Manual Trigger                | Initiate workflow execution manually             |                           | Configuration            |                                                                                                                              |
| Configuration            | Set                           | Define keyword parameter for patent search       | Manual Trigger            | NASA Patents API         |                                                                                                                              |
| NASA Patents API          | HTTP Request                  | Fetch patent data from NASA API                    | Configuration             | Loop Over Patents        |                                                                                                                              |
| Loop Over Patents         | SplitInBatches                | Process patents one at a time                      | NASA Patents API          | Google Search - Find Company | "### 1. Data Acquisition\nGet patent data and find potential companies."                                                    |
| Google Search - Find Company | Apify Actor                  | Search Google for companies related to patents    | Loop Over Patents         | Extract Company URL      |                                                                                                                              |
| Extract Company URL       | Code (JavaScript)             | Extract top company URL and name from search      | Google Search - Find Company | Crawl Company Website    |                                                                                                                              |
| Crawl Company Website     | Apify Actor                   | Crawl company website for business information    | Extract Company URL       | Analyze Fit & Draft Email | "### 2. Enrichment & Analysis\nCrawl websites and use AI to determine fit."                                                 |
| Analyze Fit & Draft Email | LangChain OpenAI node         | AI scoring and drafting personalized outreach email | Crawl Company Website     | High Score Filter        |                                                                                                                              |
| High Score Filter         | If node                      | Filter leads with high AI-assigned score          | Analyze Fit & Draft Email | Google Search - Find LinkedIn |                                                                                                                              |
| Google Search - Find LinkedIn | Apify Actor                  | Search LinkedIn profiles related to company       | High Score Filter         | Create Notion Lead       |                                                                                                                              |
| Create Notion Lead        | Notion node                  | Save enriched lead data into Notion database      | Google Search - Find LinkedIn |                          |                                                                                                                              |
| Template Description      | Sticky Note                  | Workflow general description and usage guide      |                           |                          | See detailed content under section 5                                                                                         |
| Step 1                    | Sticky Note                  | Label for Data Acquisition block                   |                           |                          |                                                                                                                              |
| Step 2                    | Sticky Note                  | Label for Enrichment & Analysis block              |                           |                          |                                                                                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow manually  

2. **Create Set Node: Configuration**  
   - Type: Set  
   - Add field: `keyword` (string)  
   - Default value: `"robotics"` (modifiable)  
   - Connect Manual Trigger output to this node  

3. **Create HTTP Request Node: NASA Patents API**  
   - Type: HTTP Request  
   - Method: GET  
   - URL:  
     `https://api.nasa.gov/techtransfer/patent/?engine&query={{ $json.keyword }}&api_key=<NASA_API_KEY>`  
   - Replace `<NASA_API_KEY>` with your actual NASA API key  
   - Connect Configuration output to this node  

4. **Create SplitInBatches Node: Loop Over Patents**  
   - Type: SplitInBatches  
   - Default batch size (1)  
   - Connect NASA Patents API output to this node  

5. **Create Apify Node: Google Search - Find Company**  
   - Type: Apify (Run Actor)  
   - Resource: Actors  
   - Operation: Run actor  
   - Actor: Use `google-search-scraper` or equivalent  
   - Input: Pass patent title or relevant keyword from current batch item  
   - Connect Loop Over Patents output to this node  
   - Configure Apify credentials  

6. **Create Code Node: Extract Company URL**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     const results = items[0].json.organicResults;
     return results && results.length > 0 ? [{ json: { targetUrl: results[0].url, companyName: results[0].title } }] : [];
     ```  
   - Connect Google Search - Find Company output to this node  

7. **Create Apify Node: Crawl Company Website**  
   - Type: Apify (Run Actor)  
   - Resource: Actors  
   - Operation: Run actor  
   - Actor: Use `website-content-crawler` or equivalent  
   - Input: `targetUrl` from Extract Company URL node  
   - Connect Extract Company URL output to this node  
   - Configure Apify credentials  

8. **Create LangChain OpenAI Node: Analyze Fit & Draft Email**  
   - Type: LangChain OpenAI  
   - Operation: message (chat completion)  
   - Input: Combine patent and company website content data  
   - Connect Crawl Company Website output to this node  
   - Configure OpenAI credentials  

9. **Create If Node: High Score Filter**  
   - Type: If  
   - Condition: `$json.score` matches regex `S|A`  
   - Connect Analyze Fit & Draft Email output to this node  

10. **Create Apify Node: Google Search - Find LinkedIn**  
    - Type: Apify (Run Actor)  
    - Resource: Actors  
    - Operation: Run actor  
    - Actor: Use Google search actor focused on LinkedIn profiles  
    - Connect True output from High Score Filter to this node  
    - Configure Apify credentials  

11. **Create Notion Node: Create Notion Lead**  
    - Type: Notion  
    - Resource: Database Page  
    - Database ID: Enter your Notion database ID for startup leads  
    - Map fields as follows:  
      - Company (rich_text): from Extract Company URL `companyName`  
      - Website (url): from Extract Company URL `targetUrl`  
      - LinkedIn (url): from Google Search - Find LinkedIn results  
      - Email (email): from Analyze Fit & Draft Email `email`  
      - Score (number): from Analyze Fit & Draft Email `score`  
      - Draft Email (rich_text): from Analyze Fit & Draft Email `draft_email`  
      - NASA Tech (rich_text): from Loop Over Patents patent title  
    - Connect Google Search - Find LinkedIn output to this node  
    - Configure Notion OAuth2 credentials  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                       | Context or Link                                                        |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| 🚀 Automated NASA Tech Scout & Outreach Generator: This workflow automates finding commercial partners for NASA patents by scouting startups, analyzing fit via AI, and preparing outreach emails. Designed for Innovation Managers, Tech Transfer Offices, and Business Development reps.                                                                 | See Template Description sticky note within the workflow              |
| Setup Instructions: 1. Define search keyword in Configuration node. 2. Insert NASA API key in NASA Patents API node (get from api.nasa.gov). 3. Connect Apify account and ensure access to `google-search-scraper` and `website-content-crawler` actors. 4. Connect OpenAI API key. 5. Create Notion database with required schema and enter its ID. | See Template Description sticky note content                           |
| Notion Database Schema: Company (Text), Website (URL), LinkedIn (URL), Email (Email), Score (Number), Draft Email (Text), NASA Tech (Text).                                                                                                                                                                                                        | For proper data saving in "Create Notion Lead" node                   |
| Apify Actors Required: `google-search-scraper` for Google searches; `website-content-crawler` for crawling company websites.                                                                                                                                                                                                                       | Requires Apify account and correct actor usage                        |
| OpenAI Usage: The workflow uses OpenAI chat completions to analyze company fit and draft personalized outreach emails, requiring OpenAI credentials with sufficient quota.                                                                                                                                                                          | LangChain OpenAI node configuration                                   |

---

*Disclaimer: The provided text is extracted exclusively from an automated workflow created with n8n, compliant with current content policies and containing no illegal, offensive, or protected elements. All manipulated data is legal and public.*