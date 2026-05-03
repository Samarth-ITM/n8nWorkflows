Automating Job Searches on LinkedIn and X, then Saving Results to Notion

https://n8nworkflows.xyz/workflows/automating-job-searches-on-linkedin-and-x--then-saving-results-to-notion-9793


# Automating Job Searches on LinkedIn and X, then Saving Results to Notion

### 1. Workflow Overview

This workflow automates the daily job search process focused on senior designer roles on two platforms: LinkedIn and X (formerly Twitter). It scrapes public job postings, parses relevant details, filters based on specified criteria, and saves qualified listings to a Notion database for easy tracking and management.

The workflow is logically divided into two main parallel branches reflecting the two platforms and their processing pipelines:

- **1.1 LinkedIn Job Search Automation**  
  Scheduled daily at 5 AM, it searches LinkedIn job listings with customized keywords and locations, extracts job details from the HTML of search results, fetches full job descriptions and poster info from individual job pages, filters incomplete entries, waits between requests to avoid rate limiting, and saves validated jobs to Notion.

- **1.2 X (Twitter) Job Search Automation**  
  Scheduled daily at 5:15 AM, it searches for relevant job-related tweets containing senior designer roles using complex search queries, parses tweet content and metadata to extract job and poster details, filters out unwanted listings and locations, waits to manage rate limits, and saves results to the same Notion database.

Each branch includes nodes for search criteria setup, HTTP requests or API calls, data parsing and filtering, rate limiting handling, and saving to Notion. The workflow includes multiple sticky notes providing usage guidance, customization tips, and warnings about potential HTML structure changes on LinkedIn.

---

### 2. Block-by-Block Analysis

#### 2.1 LinkedIn Job Search Automation

- **Overview:**  
  This block automatically searches LinkedIn for senior designer jobs every day at 5 AM using user-defined keywords and location filters. It parses the job listings page HTML to extract job cards, filters out incomplete data, waits between requests to prevent rate limiting, fetches detailed job descriptions and poster information from each job’s page, and finally saves the data to Notion.

- **Nodes Involved:**  
  - Everyday @5am1 (Schedule Trigger)  
  - Set Search Criteria1 (Set)  
  - Search LinkedIn1 (HTTP Request)  
  - Parse Jobs1 (Code)  
  - Limit1 (Limit)  
  - Filter1 (Filter)  
  - Wait (Wait)  
  - Fetch Job Details1 (HTTP Request)  
  - Extract Poster Info2 (Code)  
  - Save to Notion3 (Notion)  
  - Sticky Notes: BookSlot Webhook1, BookSlot Webhook2, Sticky Note, Sticky Note1, Sticky Note2, Notes

- **Node Details:**

  - **Everyday @5am1**  
    - *Type:* Schedule Trigger  
    - *Role:* Triggers workflow daily at 5 AM to start LinkedIn job search.  
    - *Configuration:* Default daily interval, no complex cron.  
    - *Input:* None  
    - *Output:* Triggers Set Search Criteria1  
    - *Failure Cases:* Trigger misconfiguration or n8n instance downtime prevents start.

  - **Set Search Criteria1**  
    - *Type:* Set  
    - *Role:* Defines search parameters such as job titles, excluded keywords, locations, time filters, and sorting for LinkedIn search.  
    - *Configuration:*  
      - search_keywords: "senior product designer, product design lead, senior UX designer, AI designer"  
      - excluded_keywords: "contract, freelance"  
      - location: "remote, san francisco"  
      - f_TPR (time posted range): "r86400" (last 24h)  
      - sortBy: "DD" (most recent first)  
    - *Input:* Trigger output  
    - *Output:* Passes parameters to Search LinkedIn1  
    - *Failure Cases:* Incorrect parameter formatting could lead to invalid LinkedIn query URLs.

  - **Search LinkedIn1**  
    - *Type:* HTTP Request  
    - *Role:* Performs HTTP GET request to LinkedIn job search page with encoded search criteria embedded in URL.  
    - *Configuration:* URL constructed dynamically using set parameters (hardcoded in node JSON here for demo)  
    - *Response Format:* String (HTML)  
    - *Input:* Search criteria from Set Search Criteria1  
    - *Output:* Raw HTML of job search results to Parse Jobs1  
    - *Failure Cases:* HTTP errors, LinkedIn IP blocking, HTML structure changes, rate limiting.

  - **Parse Jobs1**  
    - *Type:* Code  
    - *Role:* Parses LinkedIn job search page HTML to extract job URLs, titles, companies, locations, and posting dates. Uses regex matching over HTML snippets.  
    - *Key Logic:*  
      - Extracts multiple job URLs from href attributes.  
      - For each URL, extracts job title, company (multiple fallback regex patterns), location, and posted date from surrounding HTML.  
    - *Input:* HTML string from Search LinkedIn1  
    - *Output:* List of jobs with parsed metadata to Limit1  
    - *Failure Cases:* Regex failures if LinkedIn changes HTML structure, missing data fields.

  - **Limit1**  
    - *Type:* Limit  
    - *Role:* Restricts the number of job entries processed downstream to 10 per run to avoid excessive requests.  
    - *Configuration:* maxItems = 10  
    - *Input:* Parsed jobs from Parse Jobs1  
    - *Output:* Limited jobs to Filter1  
    - *Failure Cases:* None significant; misconfiguration limits data volume.

  - **Filter1**  
    - *Type:* Filter  
    - *Role:* Filters out job entries missing critical info: excludes jobs with company or job title equal to "Company not found" or "Title not found".  
    - *Input:* Limited job list from Limit1  
    - *Output:* Filtered valid jobs to Wait  
    - *Failure Cases:* Overly strict filtering may exclude valid entries if parsing is imperfect.

  - **Wait**  
    - *Type:* Wait  
    - *Role:* Adds a 10-second delay between requests to LinkedIn to reduce risk of being rate limited or blocked.  
    - *Configuration:* 10 seconds  
    - *Input:* Filtered jobs from Filter1  
    - *Output:* To Fetch Job Details1  
    - *Failure Cases:* Workflow slowdown; no critical failure.

  - **Fetch Job Details1**  
    - *Type:* HTTP Request  
    - *Role:* Fetches full HTML content of each individual LinkedIn job posting page using job_url.  
    - *Input:* Job URLs from Wait  
    - *Output:* HTML content of job detail pages to Extract Poster Info2  
    - *Failure Cases:* HTTP errors, access restrictions, LinkedIn blocking, dynamic content not loaded in static HTML.

  - **Extract Poster Info2**  
    - *Type:* Code  
    - *Role:* Parses job detail HTML to extract job description (cleaned and truncated to 2000 chars), poster's name, title, and LinkedIn profile URL using regex.  
    - *Key Expressions:*  
      - Description: Matches 'show-more-less-html__markup' div, cleans HTML tags.  
      - Poster name/title/profile: Matches classes related to hiring team member.  
    - *Input:* Job detail HTML from Fetch Job Details1  
    - *Output:* Enhanced job data with poster info to Save to Notion3  
    - *Failure Cases:* HTML changes break parsing, missing poster info, description truncation.

  - **Save to Notion3**  
    - *Type:* Notion  
    - *Role:* Creates a new page in specified Notion database for each job with mapped properties: Job Title, Company, Location, Job URL, Poster Name, Poster Title, Poster Profile, Job Description.  
    - *Configuration:* Uses Notion credentials and databaseId (set via credentials UI).  
    - *Input:* Fully parsed job data with poster info from Extract Poster Info2  
    - *Output:* None (endpoint)  
    - *Failure Cases:* Notion API authentication errors, rate limits, database schema mismatch.

  - **Sticky Notes**  
    - Provide comprehensive documentation on goal, setup, customization, notes about LinkedIn scraping risks, and usage instructions.

---

#### 2.2 X (Twitter) Job Search Automation

- **Overview:**  
  This block triggers daily at 5:15 AM, searches X for tweets containing senior designer job openings matching specified keywords and locations, parses tweets to extract job, poster, and metadata, filters out irrelevant or incomplete posts, waits to comply with rate limits, and saves results to Notion.

- **Nodes Involved:**  
  - Everyday @5:15am (Schedule Trigger)  
  - Set Search (Set)  
  - Search Twitter Job Posts (Twitter)  
  - Parse and Filter Jobs (Code)  
  - Filter Valid Jobs (If)  
  - Wait. (Wait)  
  - Save to Notion Database (Notion)  
  - Sticky Note BookSlot Webhook7

- **Node Details:**

  - **Everyday @5:15am**  
    - *Type:* Schedule Trigger  
    - *Role:* Starts this branch daily at 5:15 AM.  
    - *Input:* None  
    - *Output:* To Set Search  

  - **Set Search**  
    - *Type:* Set  
    - *Role:* Defines Twitter search parameters: job titles, excluded keywords, locations, posting date range.  
    - *Configuration:*  
      - search_keywords: "senior product designer, product design lead, senior UX designer"  
      - excluded_keywords: "junior, intern, internship"  
      - location: "san francisco, remote"  
      - posted_within: "24h"  
    - *Input:* Trigger output  
    - *Output:* To Search Twitter Job Posts  

  - **Search Twitter Job Posts**  
    - *Type:* Twitter  
    - *Role:* Executes a Twitter API search query combining job titles, hiring phrases, locations, and excluding contract/freelance/junior/intern posts and retweets.  
    - *Configuration:* Uses OAuth2 credentials for X account, search text is a complex boolean query string.  
    - *Input:* Search parameters from Set Search (though parameters are hardcoded in node here)  
    - *Output:* Tweets matching query to Parse and Filter Jobs  
    - *Failure Cases:* Twitter API rate limits, OAuth token expiration, API changes.

  - **Parse and Filter Jobs**  
    - *Type:* Code  
    - *Role:* Parses tweets and metadata to extract job title, company name, location, job URLs, poster info (name, username, profile, bio), and filters out tweets containing excluded keywords or non-SF/remote locations.  
    - *Key Logic:*  
      - Uses regex patterns to find job titles and company names in tweet text.  
      - Attempts to infer company from tweet author bio if not in text.  
      - Extracts location with priority to San Francisco and Remote mentions.  
      - Builds tweet URL from author username and tweet ID.  
      - Extracts poster title from bio text.  
      - Limits description to 2000 characters.  
    - *Input:* Tweets from Search Twitter Job Posts  
    - *Output:* Parsed job objects to Filter Valid Jobs  
    - *Failure Cases:* Regex misses, changed tweet structures, incomplete author info.

  - **Filter Valid Jobs**  
    - *Type:* If  
    - *Role:* Ensures only jobs with non-empty job_title and company fields proceed.  
    - *Input:* Parsed tweets from previous node  
    - *Output:* Valid jobs to Wait.  
    - *Failure Cases:* Strict filtering may exclude borderline valid jobs.

  - **Wait.**  
    - *Type:* Wait  
    - *Role:* Adds delay (default unspecified, likely minimal) to avoid API rate limits.  
    - *Input:* Filtered valid jobs  
    - *Output:* To Save to Notion Database  

  - **Save to Notion Database**  
    - *Type:* Notion  
    - *Role:* Creates entries in Notion database with fields: Job Title, Company, Location, Job URL, Poster Name, Poster Title, Poster Profile, Job Description, Tweet URL, Source, Posted Date.  
    - *Configuration:* Uses Notion OAuth credentials and databaseId set in node UI.  
    - *Input:* Parsed and validated tweets from Wait.  
    - *Failure Cases:* Notion API errors, schema mismatch, rate limits.

  - **Sticky Note BookSlot Webhook7**  
    - Provides goal context specifically for X job search focusing on senior designer roles linking directly to posts.

---

### 3. Summary Table

| Node Name                | Node Type          | Functional Role                                      | Input Node(s)            | Output Node(s)                | Sticky Note                                                                                   |
|--------------------------|--------------------|-----------------------------------------------------|--------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| Sticky Note BookSlot Webhook1 | Sticky Note       | Describes goal: search only Senior Designer roles on LinkedIn | None                     | None                         | # Goal: Search only Senior Designer roles on LinkedIn (customize included/excluded job titles) |
| Sticky Note BookSlot Webhook2 | Sticky Note       | Setup instructions and general LinkedIn job search info | None                     | None                         | See detailed setup instructions including Notion connection, customization, schedule, limits |
| Sticky Note BookSlot Webhook7 | Sticky Note       | Goal for X job search branch                         | None                     | None                         | # Goal: Search only Senior Designer jobs from X and link directly to each post                 |
| Sticky Note               | Sticky Note       | Workflow process summary for LinkedIn branch        | None                     | None                         | Explains workflow steps: trigger, search, parse, filter, wait, fetch details, save to Notion  |
| Sticky Note1              | Sticky Note       | Customization instructions for search keywords      | None                     | None                         | #CUSTOMIZE YOUR SEARCH - Edit values to match job preferences                                 |
| Sticky Note2              | Sticky Note       | Notes about LinkedIn scraping risks and filtering   | None                     | None                         | Notes on LinkedIn scraping risks, HTML changes, filtering out jobs missing title or company   |
| Everyday @5am1            | Schedule Trigger  | Triggers LinkedIn job search daily at 5 AM          | None                     | Set Search Criteria1          |                                                                                                |
| Set Search Criteria1      | Set               | Defines LinkedIn search parameters                   | Everyday @5am1           | Search LinkedIn1              |                                                                                                |
| Search LinkedIn1          | HTTP Request      | Fetches LinkedIn job search results HTML             | Set Search Criteria1     | Parse Jobs1                  |                                                                                                |
| Parse Jobs1               | Code              | Parses LinkedIn HTML for job cards                   | Search LinkedIn1         | Limit1                       |                                                                                                |
| Limit1                   | Limit             | Limits jobs processed to max 10                      | Parse Jobs1              | Filter1                      |                                                                                                |
| Filter1                  | Filter            | Filters out jobs missing title or company            | Limit1                   | Wait                         |                                                                                                |
| Wait                     | Wait              | Adds 10 second delay between LinkedIn requests      | Filter1                  | Fetch Job Details1            |                                                                                                |
| Fetch Job Details1        | HTTP Request      | Fetches individual job posting pages                 | Wait                     | Extract Poster Info2          |                                                                                                |
| Extract Poster Info2      | Code              | Extracts job description and poster info from HTML  | Fetch Job Details1       | Save to Notion3               |                                                                                                |
| Save to Notion3          | Notion            | Saves LinkedIn job data to Notion database           | Extract Poster Info2     | None                         |                                                                                                |
| Everyday @5:15am          | Schedule Trigger  | Triggers X job search daily at 5:15 AM               | None                     | Set Search                   |                                                                                                |
| Set Search                | Set               | Defines Twitter search parameters                     | Everyday @5:15am         | Search Twitter Job Posts      |                                                                                                |
| Search Twitter Job Posts  | Twitter            | Searches tweets for job posts                         | Set Search               | Parse and Filter Jobs         |                                                                                                |
| Parse and Filter Jobs     | Code              | Parses tweets and filters job postings                | Search Twitter Job Posts | Filter Valid Jobs             |                                                                                                |
| Filter Valid Jobs         | If                | Filters tweets with non-empty job title and company  | Parse and Filter Jobs    | Wait.                        |                                                                                                |
| Wait.                    | Wait              | Waits between Twitter processing steps                | Filter Valid Jobs        | Save to Notion Database       |                                                                                                |
| Save to Notion Database   | Notion            | Saves X job data to Notion database                   | Wait.                    | None                         |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the LinkedIn Job Search Branch**

   1. Add a **Schedule Trigger** node named `Everyday @5am1` set to trigger daily at 5:00 AM.

   2. Add a **Set** node named `Set Search Criteria1` connected from the trigger. Configure string fields:  
      - `search_keywords` = "senior product designer, product design lead, senior UX designer, AI designer"  
      - `excluded_keywords` = "contract, freelance"  
      - `location` = "remote, san francisco"  
      - `f_TPR` = "r86400" (last 24h)  
      - `sortBy` = "DD" (most recent first)

   3. Add an **HTTP Request** node named `Search LinkedIn1` connected from `Set Search Criteria1`. Configure:  
      - Method: GET  
      - URL: Construct a LinkedIn jobs search URL using the parameters, e.g.,  
        `https://www.linkedin.com/jobs/search?keywords=senior%20product%20designer%20OR%20product%20design%20lead%20OR%20senior%20UX%20designer&location=san%20francisco&f_TPR=r86400`  
      - Response format: String (HTML)  
      - No authentication needed.

   4. Add a **Code** node named `Parse Jobs1` connected from `Search LinkedIn1`. Paste the JavaScript code that parses the HTML for job URLs, titles, companies, locations, and posted dates using regex.

   5. Add a **Limit** node named `Limit1` connected from `Parse Jobs1`. Set max items to 10 to limit job processing per run.

   6. Add a **Filter** node named `Filter1` connected from `Limit1`. Configure to exclude any job where `company` equals "Company not found" or `job_title` equals "Title not found".

   7. Add a **Wait** node named `Wait` connected from `Filter1`. Set wait time to 10 seconds.

   8. Add an **HTTP Request** node named `Fetch Job Details1` connected from `Wait`. Configure:  
      - Method: GET  
      - URL: Use expression `{{$json.job_url}}` to fetch each job’s detail page.  
      - Response format: Default (string)

   9. Add a **Code** node named `Extract Poster Info2` connected from `Fetch Job Details1`. Paste the JavaScript code that extracts job description (cleaned and truncated), poster name, title, and LinkedIn profile URL from the job detail page HTML.

   10. Add a **Notion** node named `Save to Notion3` connected from `Extract Poster Info2`. Configure:  
       - Resource: Database Page  
       - Database ID: Select your Notion job listings database  
       - Map properties:  
         - Job Title → `job_title`  
         - Company → `company`  
         - Location → `location`  
         - Job URL → `job_url`  
         - Poster Name → `poster_name`  
         - Poster Title → `poster_title`  
         - Poster Profile → `poster_profile`  
         - Job Description → `description`  
       - Set up Notion credentials with OAuth2.

2. **Create the X (Twitter) Job Search Branch**

   1. Add a **Schedule Trigger** node named `Everyday @5:15am` set to trigger daily at 5:15 AM.

   2. Add a **Set** node named `Set Search` connected from the trigger. Configure string fields:  
      - `search_keywords` = "senior product designer, product design lead, senior UX designer"  
      - `excluded_keywords` = "junior, intern, internship"  
      - `location` = "san francisco, remote"  
      - `posted_within` = "24h"

   3. Add a **Twitter** node named `Search Twitter Job Posts` connected from `Set Search`. Configure:  
      - Operation: Search  
      - Search Text:  
        ```  
        ("senior product designer" OR "product design lead" OR "senior UX designer") (hiring OR "we're hiring" OR "join our team" OR opening OR "we are hiring") ("san francisco" OR remote OR "work from anywhere") -contract -freelance -junior -intern -internship -RT  
        ```  
      - Credentials: OAuth2 configured for your X account.

   4. Add a **Code** node named `Parse and Filter Jobs` connected from `Search Twitter Job Posts`. Paste JavaScript code which:  
      - Parses tweets for job titles, company names, locations (favoring SF/remote), and URLs.  
      - Filters out excluded keywords and irrelevant locations.  
      - Extracts poster info and tweet metrics.  
      - Limits description to 2000 characters.

   5. Add an **If** node named `Filter Valid Jobs` connected from `Parse and Filter Jobs`. Configure to pass only if `job_title` and `company` are not empty.

   6. Add a **Wait** node named `Wait.` connected from `Filter Valid Jobs`. No wait time is mandatory but recommended to avoid Twitter API limits.

   7. Add a **Notion** node named `Save to Notion Database` connected from `Wait.`. Configure:  
      - Resource: Database Page  
      - Database ID: Same Notion database as LinkedIn branch  
      - Map properties:  
        - Job Title → `job_title`  
        - Company → `company`  
        - Location → `location`  
        - Job URL → `job_url`  
        - Poster Name → `poster_name`  
        - Poster Title → `poster_title`  
        - Poster Profile → `poster_profile`  
        - Job Description → `description`  
        - Tweet URL → `tweet_url`  
        - Source → `source` (select type: twitter)  
        - Posted Date → `posted_date`  
      - Use existing Notion OAuth credentials.

3. **Add Sticky Notes** at appropriate places to document goals, setup instructions, customization tips, and warnings about scraping reliability and rate limits.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------------|
| Creator: [Summer Chang YouTube Channel](https://www.youtube.com/channel/UCAdp-nOSH-jcrwXkLlUMyXQ) — The workflow is originally created and shared by Summer Chang.                                                                                                                                                            | Creator credit                                                                                                                               |
| Setup instructions include creating or duplicating a Notion database template: [Notion Job Search Automation Template](https://summerchangco.notion.site/job-search-automation?v=28e2d5cd4ef48197a875000cb99628e5&source=copy_link)                                                                                                | Notion database template for job listings                                                                                                   |
| LinkedIn scraping depends on the current HTML structure, which may change and break parsing logic. Use caution respecting LinkedIn’s Terms of Service and avoid aggressive scraping to prevent IP blocking.                                                                                                                    | Notes on LinkedIn scraping risks                                                                                                            |
| The workflow includes delays (wait nodes) to reduce rate limiting risks on LinkedIn and Twitter APIs. Adjust delays if rate limits or blocks occur.                                                                                                                                                                         | Rate limiting and wait time advice                                                                                                          |
| Twitter search is done via official Twitter API (OAuth2) and uses a complex boolean search query tuned for senior designer roles in San Francisco or remote jobs, excluding junior/internship and contract roles.                                                                                                              | Twitter API search details                                                                                                                  |
| Notion integration requires API access and proper database setup matching the expected schema with properties such as Job Title (title), Company (rich_text), Location, Job URL, Poster info, Description, etc.                                                                                                               | Notion API and database setup instructions                                                                                                 |

---

This detailed reference document enables advanced users or AI agents to fully understand, reproduce, and modify the workflow, as well as anticipate potential integration issues and failure points.