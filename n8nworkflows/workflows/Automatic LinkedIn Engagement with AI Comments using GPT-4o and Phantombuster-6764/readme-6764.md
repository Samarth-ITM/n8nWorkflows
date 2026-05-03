Automatic LinkedIn Engagement with AI Comments using GPT-4o and Phantombuster

https://n8nworkflows.xyz/workflows/automatic-linkedin-engagement-with-ai-comments-using-gpt-4o-and-phantombuster-6764


# Automatic LinkedIn Engagement with AI Comments using GPT-4o and Phantombuster

---

### 1. Workflow Overview

This workflow automates LinkedIn engagement by generating AI-crafted comments on LinkedIn posts discovered by keyword searches, then posting those comments automatically. It leverages GPT-4o for natural language generation, Phantombuster agents for LinkedIn data scraping and auto-commenting, and Microsoft SharePoint for managing session cookies, CSV files, and deduplication lists.

The workflow is logically divided into the following blocks:

- **1.1 Cookie & Search Term Selection:** Downloads session cookies, selects an active cookie based on current time, and generates a random LinkedIn search keyword using GPT-4o.
- **1.2 LinkedIn Posts Scraping:** Launches a Phantombuster Search Agent to scrape LinkedIn posts matching the search term, then retrieves the posts.
- **1.3 Comment Generation:** Picks a random post from the scraped data and uses GPT-4o to generate a LinkedIn-optimized comment in a specified language and style.
- **1.4 Deduplication Check:** Downloads a CSV of already-commented posts, checks if the selected post has been commented on before, and updates the CSV if new.
- **1.5 Comment Posting Automation:** Creates a CSV with the post URL and comment, uploads it to SharePoint, and triggers Phantombuster’s Auto-comment Agent to post the comment on LinkedIn.
- **1.6 Rate Limiting & Scheduling:** Controls execution frequency with scheduling and wait nodes to comply with LinkedIn and API rate limits.

---

### 2. Block-by-Block Analysis

#### 1.1 Cookie & Search Term Selection

**Overview:**  
This block handles session authentication by selecting an appropriate session cookie based on time slices and generates a random, realistic LinkedIn search term related to AI and automation.

**Nodes Involved:**  
- Schedule Trigger  
- Get Available Session Cookies  
- Extract Cookies  
- Select Cookie (GPT-4o agent)  
- OpenAI Chat Model2 (GPT-4o)  
- Generate Random Search Term (GPT-4o agent)  
- OpenAI Chat Model1 (GPT-4o)  
- Set ENV Variables  

**Node Details:**

- **Schedule Trigger**  
  - Type: ScheduleTrigger  
  - Role: Starts workflow hourly to maintain posting cadence.  
  - Config: Interval set to every hour.  
  - Inputs: None (trigger node)  
  - Outputs: Connects to `Get Available Session Cookies`.  
  - Edge Cases: Missed triggers or overlapping runs could cause rate limit issues.

- **Get Available Session Cookies**  
  - Type: Microsoft SharePoint  
  - Role: Downloads a text file containing LinkedIn session cookies from SharePoint.  
  - Config: Points to a specific SharePoint site, folder, and file (linkedin_session_cookies.txt).  
  - Inputs: From `Schedule Trigger`.  
  - Outputs: Connects to `Extract Cookies`.  
  - Edge Cases: File not found or SharePoint auth failure.

- **Extract Cookies**  
  - Type: ExtractFromFile (text operation)  
  - Role: Extracts raw cookie strings from the downloaded file content.  
  - Inputs: From `Get Available Session Cookies`.  
  - Outputs: Connects to `Select Cookie`.  
  - Edge Cases: Empty or malformed file content.

- **Select Cookie**  
  - Type: LangChain Agent (GPT-4o)  
  - Role: Selects one session cookie based on current Berlin time, dividing the day into slices equal to the number of cookies.  
  - Config: Prompt includes logic for cookie selection by time slices and returns a JSON with the selected cookie.  
  - Inputs: Raw cookie list from `Extract Cookies`.  
  - Outputs: Connects to `Generate Random Search Term`.  
  - Edge Cases: Incorrect time zone, unexpected cookie list length, or agent output format issues.

- **OpenAI Chat Model2**  
  - Type: LangChain OpenAI Chat Model (GPT-4o)  
  - Role: Runs the GPT-4o model to support the `Select Cookie` node (used implicitly by LangChain agent).  
  - Inputs: From `Select Cookie` (ai_languageModel input).  
  - Outputs: Back to `Select Cookie`.  
  - Edge Cases: API key invalid or rate limiting.

- **Generate Random Search Term**  
  - Type: LangChain Agent  
  - Role: Asks GPT-4o to generate a realistic, random LinkedIn search term related to AI/business automation. Returns a JSON object with the search term.  
  - Inputs: From `Select Cookie`.  
  - Outputs: Connects to `Set ENV Variables`.  
  - Edge Cases: Unexpected output format or irrelevant terms.

- **OpenAI Chat Model1**  
  - Type: LangChain OpenAI Chat Model (GPT-4o)  
  - Role: GPT-4o chat model used by `Generate Random Search Term`.  
  - Inputs: From `Generate Random Search Term` (ai_languageModel input).  
  - Outputs: Back to `Generate Random Search Term`.  

- **Set ENV Variables**  
  - Type: Set  
  - Role: Stores environment variables for session cookie, search term, company ID, comment prompt, comment language, and search results count.  
  - Config: Assigns variables using expressions from previous nodes.  
  - Inputs: From `Generate Random Search Term`.  
  - Outputs: Connects to `Get Search Agent`.  
  - Edge Cases: Incorrect expression evaluation or missing variables.

---

#### 1.2 LinkedIn Posts Scraping

**Overview:**  
This block launches a Phantombuster agent to scrape LinkedIn posts based on the generated search term and retrieves the raw posts data.

**Nodes Involved:**  
- Get Search Agent (Phantombuster)  
- Launch Agent (Phantombuster)  
- Wait  
- Get Posts (Phantombuster)  

**Node Details:**

- **Get Search Agent**  
  - Type: Phantombuster (get)  
  - Role: Retrieves the current configuration of the LinkedIn Search Agent to verify settings or status.  
  - Inputs: From `Set ENV Variables`.  
  - Outputs: Connects to `Launch Agent`.  
  - Edge Cases: API errors, agent not found.

- **Launch Agent**  
  - Type: Phantombuster (launch)  
  - Role: Starts the LinkedIn Search Agent with parameters including session cookie, search term, search filters, and result limits.  
  - Config: Uses environment variables to set agent arguments, including session cookie and number of results per launch.  
  - Inputs: From `Get Search Agent`.  
  - Outputs: Connects to `Wait`.  
  - Edge Cases: Agent launch failure, invalid session cookie, API quota exceeded.

- **Wait**  
  - Type: Wait  
  - Role: Waits 30 seconds after launching the agent to allow data scraping.  
  - Inputs: From `Launch Agent`.  
  - Outputs: Connects to `Get Posts`.  
  - Edge Cases: Waiting too little or too long could cause data inconsistency or delay.

- **Get Posts**  
  - Type: Phantombuster (getOutput)  
  - Role: Retrieves the output (scraped LinkedIn posts) from the Search Agent launch.  
  - Inputs: From `Wait`.  
  - Outputs: Connects to `Wait2`.  
  - Edge Cases: Data not yet ready, API errors.

---

#### 1.3 Comment Generation

**Overview:**  
Randomly selects one post from the scraped posts and uses GPT-4o to generate a LinkedIn-appropriate comment following defined guidelines.

**Nodes Involved:**  
- Wait2  
- Get Random Post (Code)  
- Create Comment (LangChain Agent)  
- OpenAI Chat Model (GPT-4o)  
- Create CSV Binary (Code)  

**Node Details:**

- **Wait2**  
  - Type: Wait  
  - Role: Small delay (default) before processing posts to ensure data consistency.  
  - Inputs: From `Get Posts`.  
  - Outputs: Connects to `Get Random Post`.  

- **Get Random Post**  
  - Type: Code  
  - Role: Selects a random post from all scraped posts and extracts only the post URL and description (text content).  
  - Inputs: From `Wait2`.  
  - Outputs: Connects to `Download file`.  
  - Edge Cases: No posts found, empty or malformed data.

- **Create Comment**  
  - Type: LangChain Agent  
  - Role: Uses GPT-4o to generate a LinkedIn comment based on the post content and environment variables specifying comment prompt and language.  
  - Config: Prompt instructs to create a ≤150 character comment in the specified language, subtly referencing post content.  
  - Inputs: From `Get Random Post` and OpenAI Chat Model.  
  - Outputs: Connects to `Create CSV Binary`.  
  - Edge Cases: Unexpected AI output, API errors.

- **OpenAI Chat Model**  
  - Type: LangChain OpenAI Chat Model (GPT-4o)  
  - Role: The language model backend for `Create Comment`.  
  - Inputs: From `Create Comment` (ai_languageModel input).  
  - Outputs: Back to `Create Comment`.  

- **Create CSV Binary**  
  - Type: Code  
  - Role: Constructs a CSV binary buffer with one row containing the post URL (appended with company ID query) and the generated comment text, properly escaped and UTF-8 encoded.  
  - Inputs: From `Create Comment`.  
  - Outputs: Connects to `Upload CSV`.  
  - Edge Cases: Encoding issues or invalid data.

---

#### 1.4 Deduplication Check

**Overview:**  
This block prevents commenting multiple times on the same post by downloading an existing CSV list of commented posts, checking for duplicates, and updating the list if the post is new.

**Nodes Involved:**  
- Download file (CSV from SharePoint)  
- Extract from File (CSV extraction)  
- Check if in List (Code)  
- If (conditional)  
- Prepare Updated Data (Code)  
- Convert to File  
- Update file (CSV upload to SharePoint)  

**Node Details:**

- **Download file**  
  - Type: Microsoft SharePoint  
  - Role: Downloads `linkedin_posts_already_commented.csv` containing URLs of posts already commented on.  
  - Inputs: From `Get Random Post`.  
  - Outputs: Connects to `Extract from File`.  
  - Edge Cases: File missing or access denied.

- **Extract from File**  
  - Type: ExtractFromFile  
  - Role: Parses CSV with headers to output JSON items for each row.  
  - Inputs: From `Download file`.  
  - Outputs: Connects to `Check if in List`.  
  - Edge Cases: Malformed CSV, missing headers.

- **Check if in List**  
  - Type: Code  
  - Role: Normalizes URLs and checks if the selected post URL is already in the list to avoid duplicate commenting.  
  - Inputs: From `Extract from File` and `Get Random Post`.  
  - Outputs: Connects to `If` node.  
  - Edge Cases: URL mismatch due to formatting, false negatives/positives.

- **If**  
  - Type: If (Boolean condition)  
  - Role: Branches workflow based on duplicate check; if duplicate, wait; if not, proceed to update the list.  
  - Inputs: From `Check if in List`.  
  - Outputs:  
    - True (duplicate): Connects to `Wait2` (loops back to pick another post).  
    - False (new post): Connects to `Prepare Updated Data`.  

- **Prepare Updated Data**  
  - Type: Code  
  - Role: Creates a new array with existing URLs plus the new post URL for CSV update.  
  - Inputs: From `Extract from File` and `Get Random Post`.  
  - Outputs: Connects to `Convert to File`.  
  - Edge Cases: Data corruption or missing URLs.

- **Convert to File**  
  - Type: ConvertToFile  
  - Role: Converts JSON array back to CSV format for upload.  
  - Inputs: From `Prepare Updated Data`.  
  - Outputs: Connects to `Update file`.  

- **Update file**  
  - Type: Microsoft SharePoint  
  - Role: Uploads the updated CSV back to SharePoint, overwriting the old list.  
  - Inputs: From `Convert to File`.  
  - Outputs: Connects to `Create Comment`.  
  - Edge Cases: Upload failure or permission issues.

---

#### 1.5 Comment Posting Automation

**Overview:**  
Uploads the generated comment CSV to SharePoint, triggers the Phantombuster Auto-comment Agent to post the comment on LinkedIn, and waits for the posting process to complete.

**Nodes Involved:**  
- Upload CSV (Microsoft SharePoint)  
- Get Autocomment Agent (Phantombuster get)  
- Launch AC Agent (Phantombuster launch)  
- Wait1  
- Get Response (Phantombuster getOutput)  

**Node Details:**

- **Upload CSV**  
  - Type: Microsoft SharePoint  
  - Role: Uploads the one-row CSV file to SharePoint for the auto-comment agent to consume.  
  - Inputs: From `Create CSV Binary`.  
  - Outputs: Connects to `Get Autocomment Agent`.  
  - Edge Cases: Upload failure, file lock or overwrite issues.

- **Get Autocomment Agent**  
  - Type: Phantombuster (get)  
  - Role: Retrieves the auto-comment agent configuration/status.  
  - Inputs: From `Upload CSV`.  
  - Outputs: Connects to `Launch AC Agent`.  
  - Edge Cases: Agent not found or API errors.

- **Launch AC Agent**  
  - Type: Phantombuster (launch)  
  - Role: Runs the auto-comment agent with parameters including session cookie and the SharePoint CSV URL.  
  - Config: Uses environment variables and upload URL.  
  - Inputs: From `Get Autocomment Agent`.  
  - Outputs: Connects to `Wait1`.  
  - Edge Cases: Agent launch failure, invalid CSV URL.

- **Wait1**  
  - Type: Wait  
  - Role: Waits 30 seconds for the auto-comment agent to process and post the comment.  
  - Inputs: From `Launch AC Agent`.  
  - Outputs: Connects to `Get Response`.  

- **Get Response**  
  - Type: Phantombuster (getOutput)  
  - Role: Retrieves the results of the auto-comment agent run.  
  - Inputs: From `Wait1`.  
  - Outputs: End of flow or loops for next run.  
  - Edge Cases: Delayed output readiness or API timeout.

---

#### 1.6 Rate Limiting & Scheduling

**Overview:**  
Manages the pacing of the workflow execution to avoid hitting LinkedIn or API rate limits by combining hourly triggers and wait nodes between critical steps.

**Nodes Involved:**  
- Schedule Trigger (entry point)  
- Wait nodes (Wait, Wait1, Wait2)  

**Node Details:**

- **Schedule Trigger**  
  - Controls the hourly workflow start to roughly target 120 comments/day (24 runs × 5 posts each).  

- **Wait nodes**  
  - Wait after agent launches and before fetching outputs to ensure data consistency.  
  - Wait durations are 30 seconds by default but can be adjusted as needed.  
  - Edge Cases: Too short waits may cause incomplete data; too long waits reduce throughput.

---

### 3. Summary Table

| Node Name               | Node Type                    | Functional Role                           | Input Node(s)             | Output Node(s)            | Sticky Note                                                                                         |
|-------------------------|------------------------------|-----------------------------------------|---------------------------|---------------------------|---------------------------------------------------------------------------------------------------|
| Schedule Trigger        | ScheduleTrigger              | Starts workflow every hour               | —                         | Get Available Session Cookies |                                                                                                |
| Get Available Session Cookies | Microsoft SharePoint        | Downloads LinkedIn session cookies       | Schedule Trigger          | Extract Cookies           |                                                                                                |
| Extract Cookies         | ExtractFromFile (text)       | Extracts raw cookie strings               | Get Available Session Cookies | Select Cookie             |                                                                                                |
| Select Cookie           | LangChain Agent (GPT-4o)     | Selects session cookie based on time     | Extract Cookies           | Generate Random Search Term |                                                                                                |
| OpenAI Chat Model2      | LangChain OpenAI Chat Model  | GPT-4o backend for Select Cookie         | Select Cookie (ai_languageModel) | Select Cookie           |                                                                                                |
| Generate Random Search Term | LangChain Agent             | Generates random LinkedIn search term    | Select Cookie             | Set ENV Variables         |                                                                                                |
| OpenAI Chat Model1      | LangChain OpenAI Chat Model  | GPT-4o backend for Generate Random Search Term | Generate Random Search Term (ai_languageModel) | Generate Random Search Term |                                                                                                |
| Set ENV Variables       | Set                         | Stores environment variables             | Generate Random Search Term | Get Search Agent          |                                                                                                |
| Get Search Agent        | Phantombuster (get)          | Gets LinkedIn Search Agent config        | Set ENV Variables         | Launch Agent              |                                                                                                |
| Launch Agent            | Phantombuster (launch)       | Launches LinkedIn Search Agent            | Get Search Agent          | Wait                      |                                                                                                |
| Wait                    | Wait                        | Waits 30 seconds after agent launch      | Launch Agent              | Get Posts                 |                                                                                                |
| Get Posts               | Phantombuster (getOutput)    | Retrieves scraped LinkedIn posts          | Wait                      | Wait2                     |                                                                                                |
| Wait2                   | Wait                        | Waits before processing posts             | Get Posts                 | Get Random Post           |                                                                                                |
| Get Random Post         | Code                        | Selects random post from scraped data    | Wait2                     | Download file             |                                                                                                |
| Download file           | Microsoft SharePoint        | Downloads CSV of already commented posts | Get Random Post           | Extract from File         |                                                                                                |
| Extract from File       | ExtractFromFile (CSV)        | Extracts CSV data to JSON                 | Download file             | Check if in List          |                                                                                                |
| Check if in List        | Code                        | Checks if post URL already commented      | Extract from File, Get Random Post | If                     |                                                                                                |
| If                      | If (Boolean)                | Branches on duplicate check               | Check if in List          | Wait2 (if duplicate), Prepare Updated Data (if new) |                                                                                                |
| Prepare Updated Data    | Code                        | Appends new post URL to existing list    | Extract from File, Get Random Post | Convert to File          |                                                                                                |
| Convert to File         | ConvertToFile               | Converts JSON to CSV for upload           | Prepare Updated Data      | Update file               |                                                                                                |
| Update file             | Microsoft SharePoint        | Uploads updated CSV back to SharePoint   | Convert to File           | Create Comment            |                                                                                                |
| Create Comment          | LangChain Agent (GPT-4o)     | Generates LinkedIn comment from post text | Update file               | Create CSV Binary         |                                                                                                |
| OpenAI Chat Model       | LangChain OpenAI Chat Model  | GPT-4o model for Create Comment           | Create Comment (ai_languageModel) | Create Comment          |                                                                                                |
| Create CSV Binary       | Code                        | Builds CSV file with post URL and comment | Create Comment            | Upload CSV                |                                                                                                |
| Upload CSV              | Microsoft SharePoint        | Uploads CSV for auto-comment agent       | Create CSV Binary         | Get Autocomment Agent     |                                                                                                |
| Get Autocomment Agent   | Phantombuster (get)          | Gets auto-comment agent config            | Upload CSV                | Launch AC Agent           |                                                                                                |
| Launch AC Agent         | Phantombuster (launch)       | Launches auto-comment agent                | Get Autocomment Agent     | Wait1                     |                                                                                                |
| Wait1                   | Wait                        | Waits 30 seconds for auto-comment posting | Launch AC Agent           | Get Response              |                                                                                                |
| Get Response            | Phantombuster (getOutput)    | Retrieves auto-comment agent output       | Wait1                     | —                         |                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger node**  
   - Type: ScheduleTrigger  
   - Set to run every 1 hour.

2. **Create Microsoft SharePoint node “Get Available Session Cookies”**  
   - Operation: Download file  
   - Configure with SharePoint OAuth2 credentials.  
   - Point to the folder and file containing LinkedIn session cookies (linkedin_session_cookies.txt).  
   - Connect Schedule Trigger → Get Available Session Cookies.

3. **Create ExtractFromFile node “Extract Cookies”**  
   - Operation: Extract text content from downloaded file.  
   - Connect Get Available Session Cookies → Extract Cookies.

4. **Create LangChain Agent node “Select Cookie”**  
   - Configure prompt to select one cookie from the list based on current Berlin time using specified time-slicing logic.  
   - Connect Extract Cookies → Select Cookie.

5. **Create LangChain OpenAI Chat Model node “OpenAI Chat Model2”**  
   - Set model to `chatgpt-4o-latest`.  
   - Connect ai_languageModel input/output with Select Cookie.

6. **Create LangChain Agent node “Generate Random Search Term”**  
   - Prompt GPT-4o to generate a single random LinkedIn search term related to AI/business automation, outputting a raw JSON object with `search_term`.  
   - Connect Select Cookie → Generate Random Search Term.

7. **Create LangChain OpenAI Chat Model node “OpenAI Chat Model1”**  
   - Model: `chatgpt-4o-latest`.  
   - Connect ai_languageModel input/output with Generate Random Search Term.

8. **Create Set node “Set ENV Variables”**  
   - Assign variables:  
     - `ENV_SEARCH_TERM` from parsed JSON of Generate Random Search Term.  
     - `ENV_SESSION_COOKIE` from parsed JSON of Select Cookie.  
     - `ENV_SEARCH_RESULTS_PER_LAUNCH` = "5" (string).  
     - `ENV_COMMENT_PROMPT` = "Erstelle einen ansprechenden LinkedIn-Kommentar basierend auf dem gegebenen Post-Inhalt."  
     - `ENV_COMMENT_LANGUAGE` = "Deutsch"  
     - `ENV_COMPANY_ID_LINKEDIN` = "78274663"  
   - Connect Generate Random Search Term → Set ENV Variables.

9. **Create Phantombuster node “Get Search Agent”**  
   - Operation: Get agent info.  
   - Set Agent ID of LinkedIn Search Agent.  
   - Connect Set ENV Variables → Get Search Agent.

10. **Create Phantombuster node “Launch Agent”**  
    - Operation: Launch agent with JSON arguments: sessionCookie, keywords (from ENV vars), numberOfResultsPerLaunch, etc.  
    - Connect Get Search Agent → Launch Agent.

11. **Create Wait node “Wait”**  
    - Wait 30 seconds.  
    - Connect Launch Agent → Wait.

12. **Create Phantombuster node “Get Posts”**  
    - Operation: Get output of launched Search Agent.  
    - Connect Wait → Get Posts.

13. **Create Wait node “Wait2”**  
    - Default short wait.  
    - Connect Get Posts → Wait2.

14. **Create Code node “Get Random Post”**  
    - JavaScript to select a random post from all posts, returning only `postUrl` and `description`.  
    - Connect Wait2 → Get Random Post.

15. **Create Microsoft SharePoint node “Download file”**  
    - Download CSV of already-commented posts (linkedin_posts_already_commented.csv).  
    - Connect Get Random Post → Download file.

16. **Create ExtractFromFile node “Extract from File”**  
    - Extract CSV rows with headers to JSON.  
    - Connect Download file → Extract from File.

17. **Create Code node “Check if in List”**  
    - Normalize URLs and check if random post URL is already in CSV list. Output boolean `isDuplicate`.  
    - Connect Extract from File and Get Random Post → Check if in List.

18. **Create If node “If”**  
    - Condition: `isDuplicate == true`.  
    - True branch → Wait2 (retry with new post).  
    - False branch → Prepare Updated Data.  
    - Connect Check if in List → If.

19. **Create Code node “Prepare Updated Data”**  
    - Append new post URL to existing list for CSV update.  
    - Connect If (false) → Prepare Updated Data.

20. **Create ConvertToFile node “Convert to File”**  
    - Convert JSON array to CSV.  
    - Connect Prepare Updated Data → Convert to File.

21. **Create Microsoft SharePoint node “Update file”**  
    - Upload updated CSV to SharePoint (overwrite).  
    - Connect Convert to File → Update file.

22. **Create LangChain Agent node “Create Comment”**  
    - Prompt GPT-4o to generate a LinkedIn comment ≤150 chars in specified language referencing post content.  
    - Connect Update file → Create Comment.

23. **Create LangChain OpenAI Chat Model node “OpenAI Chat Model”**  
    - Model: `chatgpt-4o-latest`.  
    - Connect ai_languageModel input/output with Create Comment.

24. **Create Code node “Create CSV Binary”**  
    - Build CSV buffer with post URL + company ID and generated comment text, escaped and UTF-8 encoded.  
    - Connect Create Comment → Create CSV Binary.

25. **Create Microsoft SharePoint node “Upload CSV”**  
    - Upload one-row CSV for auto-comment agent consumption.  
    - Connect Create CSV Binary → Upload CSV.

26. **Create Phantombuster node “Get Autocomment Agent”**  
    - Get auto-comment agent configuration.  
    - Connect Upload CSV → Get Autocomment Agent.

27. **Create Phantombuster node “Launch AC Agent”**  
    - Launch auto-comment agent with session cookie and uploaded CSV URL.  
    - Connect Get Autocomment Agent → Launch AC Agent.

28. **Create Wait node “Wait1”**  
    - Wait 30 seconds to allow auto-commenting to complete.  
    - Connect Launch AC Agent → Wait1.

29. **Create Phantombuster node “Get Response”**  
    - Get output of auto-comment agent.  
    - Connect Wait1 → Get Response.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                                                  |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow designed to keep daily LinkedIn automated comments near 120/day using hourly scheduling and 30-second waits between steps.                                                                                           | Sticky Note9                                                                                   |
| Change search keywords or comment prompts by editing the `Set ENV Variables` node; language and style can be customized there as well.                                                                                        | Sticky Note4, Sticky Note6                                                                      |
| Phantombuster agents require API credentials and proper LinkedIn session cookies to operate correctly; cookies are rotated based on time slices to avoid bans.                                                                 | Sticky Note4                                                                                   |
| SharePoint is used as the file storage backend for session cookies, CSV files with post URLs, and generated comments; can be replaced by other cloud storage with node adjustments.                                            | Sticky Note7, Sticky Note8                                                                     |
| GPT-4o is used exclusively for natural language generation tasks including search term creation, session cookie selection logic, and comment generation, requiring OpenAI API credentials.                                     | Multiple nodes with LangChain GPT-4o agents                                                    |
| For improved reliability, verify that SharePoint file permissions and Phantombuster API quotas are sufficient, and monitor for API errors or timeouts that may disrupt workflow execution.                                      | General best practice                                                                           |
| Workflow logic prevents duplicate comments by checking an existing CSV of commented post URLs and updating it after each new comment is posted.                                                                              | Sticky Note8                                                                                   |
| Useful references for LinkedIn automation and Phantombuster integration can be found on the n8n community forums and official Phantombuster documentation.                                                                    | https://phantombuster.com/documentation, https://community.n8n.io/                             |

---

**Disclaimer:** The provided text is extracted exclusively from an n8n automated workflow. It adheres strictly to content policies and contains no illegal or protected elements. All data processed is legal and public.

---