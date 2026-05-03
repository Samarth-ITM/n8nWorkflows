Scheduled Instagram Auto-Liker with Phantombuster, GPT-4o & Cookie Rotation

https://n8nworkflows.xyz/workflows/scheduled-instagram-auto-liker-with-phantombuster--gpt-4o---cookie-rotation-6765


# Scheduled Instagram Auto-Liker with Phantombuster, GPT-4o & Cookie Rotation

### 1. Workflow Overview

This workflow automates Instagram post liking using a scheduled process that fetches posts by hashtags, filters duplicates, and triggers an auto-liking agent via Phantombuster. It integrates AI (GPT-4o) for hashtag generation and session cookie rotation to avoid rate limits and detection. The workflow is structured into five logical blocks:

- **1.1 Start & Session Cookie Selection:** Scheduled trigger initiates the flow, downloads session cookies, and selects one based on the current time slice.
- **1.2 Hashtag Generation and Post Scraping:** GPT-4o generates a realistic Instagram hashtag; Phantombuster agents fetch recent Instagram posts for that hashtag.
- **1.3 Duplicate Post Check:** Downloads a CSV of already liked posts from SharePoint, checks if a randomly selected post is new, and updates the CSV accordingly.
- **1.4 CSV Preparation & Auto-Like Trigger:** Creates a CSV containing the new post URL, uploads it to SharePoint, and launches a Phantombuster auto-liking agent to like the post.
- **1.5 Rate Limiting & Scheduling:** Wait nodes and schedule ensure the workflow runs every 2 hours and limits the number of likes per day to about 12.

---

### 2. Block-by-Block Analysis

#### 1.1 Start & Session Cookie Selection

- **Overview:** Starts the workflow on a 2-hour schedule, retrieves available Instagram session cookies from SharePoint, extracts and selects one cookie based on the current hour to manage session rotation.
- **Nodes Involved:**  
  - Schedule Trigger  
  - Get Available Session Cookies  
  - Extract Cookies  
  - OpenAI Chat Model1 (used to run selection prompt)  
  - Select Cookie  
  - Sticky Notes (Sticky Note4 and Sticky Note5 provide context)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Configuration: Runs every 2 hours at 15 minutes past the hour (e.g., 00:15, 02:15...)  
    - Inputs: None  
    - Outputs: Triggers next node

  - **Get Available Session Cookies**  
    - Type: Microsoft SharePoint (OAuth2)  
    - Role: Downloads the file "instagram_session_cookies.txt" from SharePoint list/folder  
    - Key config: SharePoint site and folder IDs, file name  
    - Inputs: Trigger from Schedule  
    - Outputs: Passes file content to Extract Cookies  
    - Edge cases: File missing, auth failures, network errors

  - **Extract Cookies**  
    - Type: Extract From File (text)  
    - Role: Extracts raw cookie data as text from SharePoint file  
    - Inputs: SharePoint file content  
    - Outputs: Cookie text array for selection  
    - Edge cases: Empty or malformed cookie data

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI Chat Model (chatgpt-4o-latest)  
    - Role: Runs a prompt to select the appropriate session cookie based on current Berlin time and cookie list  
    - Inputs: Cookie data, current datetime (via expressions)  
    - Outputs: JSON with selected session cookie  
    - Edge cases: API rate limits, prompt failure, unexpected output format

  - **Select Cookie**  
    - Type: LangChain Agent Node  
    - Role: Implements custom prompt logic for cookie selection: divides 24h into slices based on cookie count and picks the cookie accordingly  
    - Inputs: Cookie list text from Extract Cookies, current time context  
    - Outputs: Selected session cookie JSON  
    - Edge cases: Wrong cookie count, malformed output

- **Sticky Notes:**  
  - Sticky Note4: "Start Workflow and Retrieve Session Cookie"  
  - Sticky Note5: Explains cookie and hashtag selection logic with credentials and tweak notes

---

#### 1.2 Hashtag Generation and Post Scraping

- **Overview:** Uses GPT-4o to generate a realistic hashtag related to AI and business automation, sets environment variables, and uses Phantombuster agents to fetch Instagram posts for that hashtag.
- **Nodes Involved:**  
  - Generate Random Hashtag  
  - Set ENV Variables  
  - Get Hashtag Agent  
  - Launch Agent  
  - Get Posts  
  - Wait  
  - Sticky Notes (Sticky Note, Sticky Note6)

- **Node Details:**

  - **Generate Random Hashtag**  
    - Type: LangChain Agent Node (GPT-4o)  
    - Role: Generates one realistic hashtag related to AI/business automation  
    - Configuration: Prompt explicitly demands a raw JSON response with a single hashtag field  
    - Inputs: Selected session cookie from prior block flows via environment variables  
    - Outputs: JSON with hashtag string  
    - Edge cases: Invalid JSON response, API timeouts

  - **Set ENV Variables**  
    - Type: Set Node  
    - Role: Stores environment variables for the workflow such as:  
      - ENV_SEARCH_HASHTAGS: The generated hashtag string  
      - ENV_SESSION_COOKIE: The selected cookie value  
      - ENV_MAX_POSTS_PER_HASHTAG: Max posts to fetch (default 10)  
    - Inputs: Output from Generate Random Hashtag and Select Cookie  
    - Outputs: Variables for downstream nodes

  - **Get Hashtag Agent**  
    - Type: Phantombuster API (Agent Get)  
    - Role: Retrieves metadata about the Phantombuster agent that scrapes Instagram posts by hashtag  
    - Inputs: None except ENV variables  
    - Outputs: Agent info for launching

  - **Launch Agent**  
    - Type: Phantombuster API (Agent Launch)  
    - Role: Starts the hashtag scraping agent with arguments: maxPosts, sessionCookie, and spreadsheetUrl (hashtag search list)  
    - Inputs: Agent id from Get Hashtag Agent, environment variables  
    - Outputs: Initiates scraping

  - **Get Posts**  
    - Type: Phantombuster API (getOutput)  
    - Role: Fetches the output results (posts) of the scraping agent  
    - Inputs: Agent launched in prior node  
    - Outputs: List of Instagram posts

  - **Wait**  
    - Type: Wait Node  
    - Role: Pauses workflow to allow scraping agent time to complete (30 seconds)  
    - Inputs: Launch Agent output  
    - Outputs: Triggers Get Posts

- **Sticky Notes:**  
  - Sticky Note: "Get Instagram Posts By Custom Hashtag"  
  - Sticky Note6: Explains scraping process, Phantombuster credentials, and tweak tips

---

#### 1.3 Duplicate Post Check

- **Overview:** Processes fetched Instagram posts, selects a random post, downloads the CSV of previously liked posts from SharePoint, checks for duplicates, and branches logic accordingly.
- **Nodes Involved:**  
  - Wait2  
  - Get Random Post  
  - Download file  
  - Extract from File  
  - Check if in List  
  - If  
  - Prepare Updated Data  
  - Convert to File  
  - Update file  
  - Sticky Notes (Sticky Note1, Sticky Note7)

- **Node Details:**

  - **Wait2**  
    - Type: Wait (no parameters)  
    - Role: Small pause to ensure prior Phantombuster results are ready  
    - Inputs: Output from Get Posts or If node  
    - Outputs: Triggers Get Random Post

  - **Get Random Post**  
    - Type: Code Node  
    - Role: Selects a random Instagram post (postUrl) from the fetched posts array  
    - Important code: Throws error if no posts found; picks one randomly  
    - Inputs: Posts from Get Posts  
    - Outputs: JSON with one random post URL

  - **Download file**  
    - Type: Microsoft SharePoint (OAuth2)  
    - Role: Downloads existing CSV file "instagram_posts_already_liked.csv" listing posts already liked  
    - Inputs: Triggered after Get Random Post  
    - Outputs: Binary file content

  - **Extract from File**  
    - Type: Extract From File (CSV)  
    - Role: Extracts CSV data into JSON objects with header row recognition  
    - Inputs: Downloaded CSV binary  
    - Outputs: Array of posts already liked

  - **Check if in List**  
    - Type: Code Node  
    - Role: Checks if the random post URL is already in the liked posts list (case-insensitive, trims trailing slashes)  
    - Includes normalization and detailed logging  
    - Outputs: JSON with boolean isDuplicate flag

  - **If**  
    - Type: If Node  
    - Role: Branches workflow depending on isDuplicate flag  
    - If true (duplicate), workflow waits (Wait2) without processing further  
    - If false, continues to update CSV with new post

  - **Prepare Updated Data**  
    - Type: Code Node  
    - Role: Prepares updated JSON array combining existing liked posts plus the new random post URL  
    - Outputs: JSON array for CSV conversion

  - **Convert to File**  
    - Type: Convert To File (CSV)  
    - Role: Converts updated JSON array back to CSV binary format for upload

  - **Update file**  
    - Type: Microsoft SharePoint (OAuth2)  
    - Role: Updates existing CSV file with new content on SharePoint  
    - Inputs: CSV binary from Convert to File  
    - Outputs: Triggers CSV creation for autoliking

- **Sticky Notes:**  
  - Sticky Note1: "Process Posts and Check if not already processed"  
  - Sticky Note7: Explains duplicate check logic, credentials, and tweak notes

---

#### 1.4 CSV Preparation & Auto-Like Trigger

- **Overview:** Creates a one-row CSV with the selected post URL, uploads it to SharePoint, and launches the Phantombuster auto-liking agent to like the post.
- **Nodes Involved:**  
  - Create CSV Binary  
  - Upload CSV  
  - Get Autoliking Agent  
  - Launch AL Agent  
  - Wait1  
  - Sticky Note2, Sticky Note9

- **Node Details:**

  - **Create CSV Binary**  
    - Type: Code Node  
    - Role: Creates a CSV file with a single row containing the post URL for the autoliking agent  
    - Escapes and cleans post URL data, outputs binary with UTF-8 encoding, no BOM or header  
    - Inputs: Random post URL from Get Random Post  
    - Outputs: Binary CSV for upload

  - **Upload CSV**  
    - Type: Microsoft SharePoint (OAuth2)  
    - Role: Uploads the generated CSV file as "instagram_posts_to_like.csv" to a SharePoint folder  
    - Inputs: CSV binary from Create CSV Binary  
    - Outputs: File info including download URL for agent

  - **Get Autoliking Agent**  
    - Type: Phantombuster API (Agent Get)  
    - Role: Retrieves metadata for the autoliking agent  
    - Inputs: Triggered after CSV upload  
    - Outputs: Agent details

  - **Launch AL Agent**  
    - Type: Phantombuster API (Agent Launch)  
    - Role: Launches the autoliking agent with arguments: number of posts = 1, session cookie, and spreadsheetUrl pointing to uploaded CSV  
    - Inputs: Agent ID from Get Autoliking Agent, environment variables, uploaded CSV URL  
    - Outputs: Initiates auto-like action

  - **Wait1**  
    - Type: Wait node (30 seconds)  
    - Role: Waits for autoliking agent to complete before next iteration

- **Sticky Notes:**  
  - Sticky Note2: "Launch Autoliking Agent to Like Post"  
  - Sticky Note9: Explains CSV creation, upload, autoliking agent launch, credentials, and tweak options

---

#### 1.5 Rate Limiting & Scheduling

- **Overview:** Implements timing controls to prevent over-liking; uses schedule and wait nodes to limit likes to ~12 per day.
- **Nodes Involved:**  
  - Schedule Trigger (start node)  
  - Wait (after Launch Agent)  
  - Wait1 (after Launch AL Agent)  
  - Wait2 (after Get Posts and If node)  
  - Sticky Note8

- **Node Details:**

  - **Schedule Trigger**  
    - As above, triggers every 2 hours

  - **Wait Nodes**  
    - Wait (30s after Launch Agent): allow agent scraping to complete  
    - Wait1 (30s after Launch AL Agent): allow liking to complete  
    - Wait2 (no delay after duplicate check or posts fetch): controls pacing between steps

- **Sticky Note:**  
  - Sticky Note8: Describes scheduling and wait nodes to keep limits, tweak cron and wait durations to adjust frequency

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                  | Input Node(s)                                   | Output Node(s)                      | Sticky Note                                                                                     |
|-----------------------------|----------------------------------|-------------------------------------------------|------------------------------------------------|-----------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger                  | Initiates workflow every 2 hours                 | None                                           | Get Available Session Cookies      | ### 5) Rate Limiting & Scheduling: 2-hour cron; edit cron rule or wait durations               |
| Get Available Session Cookies| Microsoft SharePoint             | Downloads Instagram session cookies file         | Schedule Trigger                               | Extract Cookies                   | ### 1) Cookie & Hashtag Selection: Downloads cookie list; Select Cookie picks one by time slice|
| Extract Cookies             | Extract From File (text)          | Extracts raw cookie data                          | Get Available Session Cookies                   | Select Cookie                    | ### 1) Cookie & Hashtag Selection                                                              |
| OpenAI Chat Model1          | LangChain OpenAI Chat Model (GPT-4o) | Supports Select Cookie node with prompt           | Extract Cookies                                 | Select Cookie                    | ### 1) Cookie & Hashtag Selection                                                              |
| Select Cookie              | LangChain Agent                  | Selects session cookie based on current time slice | Extract Cookies, OpenAI Chat Model1             | Generate Random Hashtag            | ### 1) Cookie & Hashtag Selection                                                              |
| Generate Random Hashtag     | LangChain Agent                  | Generates realistic Instagram hashtag             | Select Cookie                                  | Set ENV Variables                | ### 1) Cookie & Hashtag Selection                                                              |
| Set ENV Variables           | Set                              | Stores environment variables for downstream nodes | Generate Random Hashtag                         | Get Hashtag Agent                |                                                                                               |
| Get Hashtag Agent           | Phantombuster API (Agent Get)    | Gets metadata of hashtag scraping agent           | Set ENV Variables                               | Launch Agent                    | ### 2) Scrape Instagram Posts                                                                  |
| Launch Agent                | Phantombuster API (Agent Launch) | Launches hashtag scraping agent                    | Get Hashtag Agent                               | Wait                           | ### 2) Scrape Instagram Posts                                                                  |
| Wait                       | Wait                             | Waits 30s for scraping to complete                 | Launch Agent                                    | Get Posts                      | ### 5) Rate Limiting & Scheduling                                                              |
| Get Posts                  | Phantombuster API (getOutput)    | Retrieves Instagram posts scraped                   | Wait                                            | Wait2                          | ### 2) Scrape Instagram Posts                                                                  |
| Wait2                      | Wait                             | Waits briefly / controls pacing                     | Get Posts, If                                    | Get Random Post, Prepare Updated Data | ### 5) Rate Limiting & Scheduling                                                              |
| Get Random Post            | Code                             | Selects a random Instagram post URL                 | Wait2                                           | Download file                  | ### 3) Duplicate Check                                                                         |
| Download file              | Microsoft SharePoint             | Downloads CSV of already liked posts                | Get Random Post                                 | Extract from File              | ### 3) Duplicate Check                                                                         |
| Extract from File          | Extract From File (CSV)          | Extracts liked posts CSV to JSON                     | Download file                                   | Check if in List              | ### 3) Duplicate Check                                                                         |
| Check if in List           | Code                             | Checks if random post URL is already liked          | Extract from File                               | If                            | ### 3) Duplicate Check                                                                         |
| If                         | If                               | Branches on isDuplicate flag                         | Check if in List                                | Wait2 (if duplicate), Prepare Updated Data (if new) | ### 3) Duplicate Check                                                                         |
| Prepare Updated Data       | Code                             | Prepares updated liked posts list with new URL       | If (false branch)                               | Convert to File               | ### 3) Duplicate Check                                                                         |
| Convert to File            | Convert To File (CSV)             | Converts updated JSON list to CSV                      | Prepare Updated Data                            | Update file                   | ### 3) Duplicate Check                                                                         |
| Update file                | Microsoft SharePoint             | Updates CSV file on SharePoint                         | Convert to File                                | Create CSV Binary             | ### 3) Duplicate Check                                                                         |
| Create CSV Binary          | Code                             | Creates one-row CSV with post URL to like             | Update file                                    | Upload CSV                   | ### 4) CSV Upload & Auto-like                                                                 |
| Upload CSV                 | Microsoft SharePoint             | Uploads CSV for autoliking agent                       | Create CSV Binary                              | Get Autoliking Agent          | ### 4) CSV Upload & Auto-like                                                                 |
| Get Autoliking Agent       | Phantombuster API (Agent Get)    | Gets metadata of autoliking agent                      | Upload CSV                                     | Launch AL Agent              | ### 4) CSV Upload & Auto-like                                                                 |
| Launch AL Agent            | Phantombuster API (Agent Launch) | Launches autoliking agent with CSV and session cookie | Get Autoliking Agent, Set ENV Variables        | Wait1                        | ### 4) CSV Upload & Auto-like                                                                 |
| Wait1                      | Wait                             | Waits 30s for auto-liking to complete                  | Launch AL Agent                                | (end or next schedule)       | ### 5) Rate Limiting & Scheduling                                                              |
| Sticky Notes (various)     | Sticky Note                      | Provide workflow documentation and context             | None                                           | None                          | See node-specific sticky note content                                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Type: Schedule Trigger  
   - Configure: Interval every 2 hours, trigger at 15 minutes past the hour  
   - No credentials

2. **Add Microsoft SharePoint Node: Get Available Session Cookies**  
   - Operation: Download file  
   - Site: plemeo.sharepoint.com (set your own)  
   - Folder: "Phantombuster" folder ID  
   - File: "instagram_session_cookies.txt"  
   - Credentials: Microsoft SharePoint OAuth2

3. **Add Extract From File Node: Extract Cookies**  
   - Operation: Text extraction  
   - Input: File from previous node

4. **Add LangChain OpenAI Chat Model Node (OpenAI Chat Model1)**  
   - Model: chatgpt-4o-latest  
   - Credentials: OpenAI API  
   - Use this node to run the cookie selection prompt (optional, can be integrated in Select Cookie)

5. **Add LangChain Agent Node: Select Cookie**  
   - Prompt: Implement the selection logic based on current Berlin time and cookie list  
   - Input: Extracted cookies text  
   - Output: JSON with "session_cookie" field

6. **Add LangChain Agent Node: Generate Random Hashtag**  
   - Prompt: Ask for a single realistic AI/business automation hashtag in raw JSON format (field "hashtag")  
   - Input: Output of Select Cookie node (for session cookie context)  
   - Credentials: OpenAI API

7. **Add Set Node: Set ENV Variables**  
   - Assign:  
     - ENV_SEARCH_HASHTAGS = `{{ $json.hashtag }}` from Generate Random Hashtag  
     - ENV_SESSION_COOKIE = `{{ $json.session_cookie }}` from Select Cookie  
     - ENV_MAX_POSTS_PER_HASHTAG = "10" (default or adjust)

8. **Add Phantombuster Node: Get Hashtag Agent**  
   - Operation: get  
   - AgentId: `4031886542434447` (hashtag scraper agent)  
   - Credentials: Phantombuster API

9. **Add Phantombuster Node: Launch Agent**  
   - Operation: launch  
   - AgentId: `={{ $json.id }}` from previous node  
   - Arguments JSON:  
     ```json
     {
       "maxPosts": "{{ $json.ENV_MAX_POSTS_PER_HASHTAG }}",
       "sessionCookie": "{{ $json.ENV_SESSION_COOKIE }}",
       "spreadsheetUrl": "{{ $json.ENV_SEARCH_HASHTAGS }}"
     }
     ```
   - Credentials: Phantombuster API

10. **Add Wait Node (Wait)**  
    - Duration: 30 seconds  
    - Connect from Launch Agent to Wait

11. **Add Phantombuster Node: Get Posts**  
    - Operation: getOutput  
    - AgentId: `4031886542434447`  
    - Credentials: Phantombuster API  
    - Connect from Wait node

12. **Add Wait Node (Wait2)**  
    - No parameters (short pause)  
    - Connect from Get Posts

13. **Add Code Node: Get Random Post**  
    - JavaScript: Select a random postUrl from input items  
    - Input: Wait2 outputs (Instagram posts list)

14. **Add Microsoft SharePoint Node: Download file**  
    - Operation: Download file  
    - File: "instagram_posts_already_liked.csv"  
    - Credentials: Microsoft SharePoint OAuth2  
    - Connect from Get Random Post

15. **Add Extract From File Node: Extract from File**  
    - Operation: CSV with header row  
    - Connect from Download file

16. **Add Code Node: Check if in List**  
    - JavaScript: Normalize URLs and check if random post URL exists in CSV data  
    - Connect from Extract from File

17. **Add If Node**  
    - Condition: `{{ $json.isDuplicate }} === true`  
    - Connect from Check if in List

18. **Add Wait Node (Wait2)**  
    - On true branch of If (duplicate) to pause and skip further processing

19. **Add Code Node: Prepare Updated Data**  
    - On false branch of If  
    - Combines existing liked posts plus new post URL as JSON array

20. **Add Convert To File Node**  
    - Converts updated JSON array to CSV  
    - Connect from Prepare Updated Data

21. **Add Microsoft SharePoint Node: Update file**  
    - Operation: Update file content  
    - File: "instagram_posts_already_liked.csv"  
    - Credentials: Microsoft SharePoint OAuth2  
    - Connect from Convert to File

22. **Add Code Node: Create CSV Binary**  
    - Creates a CSV with one row containing the new post URL  
    - Connect from Update file

23. **Add Microsoft SharePoint Node: Upload CSV**  
    - Operation: Upload file  
    - FileName: "instagram_posts_to_like.csv"  
    - Credentials: Microsoft SharePoint OAuth2  
    - Connect from Create CSV Binary

24. **Add Phantombuster Node: Get Autoliking Agent**  
    - Operation: get  
    - AgentId: `1308386990292658` (autoliking agent)  
    - Credentials: Phantombuster API  
    - Connect from Upload CSV

25. **Add Phantombuster Node: Launch AL Agent**  
    - Operation: launch  
    - AgentId: from Get Autoliking Agent  
    - Arguments JSON:  
      ```json
      {
        "numberOfPostsPerLaunch": 1,
        "sessionCookie": "{{ $json.ENV_SESSION_COOKIE }}",
        "spreadsheetUrl": "{{ $json['@content.downloadUrl'] }}"
      }
      ```  
    - Credentials: Phantombuster API  
    - Connect from Get Autoliking Agent

26. **Add Wait Node (Wait1)**  
    - Duration: 30 seconds  
    - Connect from Launch AL Agent  
    - Connect back to Schedule Trigger (for cycle)

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                       | Context or Link                                            |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Workflow uses Phantombuster API agents for Instagram scraping and auto-liking.                                                                                                                                                     | Phantombuster API documentation                             |
| Session cookie selection uses a time-slicing logic for rotating 2-4 cookies evenly over 24 hours.                                                                                                                                 | Cookie rotation logic in Select Cookie prompt               |
| GPT-4o (chatgpt-4o-latest) is used for generating realistic hashtags and for cookie selection prompt logic.                                                                                                                     | OpenAI API (LangChain node)                                 |
| SharePoint OAuth2 credentials are required to read/write CSV files storing cookies, liked posts, and upload target posts CSV.                                                                                                     | Microsoft SharePoint API                                    |
| Rate limiting achieved with schedule + wait nodes to keep likes at ~12 per day, avoiding Instagram throttling.                                                                                                                   | Sticky Note8                                                |
| CSV files structure is simple: one column "postUrl" for tracking liked posts; adjustments needed if extra metadata columns are added.                                                                                            | Duplicate check node's comment                              |
| To modify for another niche, update the Generate Random Hashtag prompt accordingly.                                                                                                                                               | Sticky Note5                                                |
| SharePoint paths and file IDs must be configured for your environment. Alternatively, replace SharePoint nodes with Dropbox or Google Drive nodes as preferred.                                                                  | Sticky Note9                                                |

---

**Disclaimer:**  
The provided text is extracted solely from an automated n8n workflow designed for Instagram auto-liking with session cookie rotation. It complies fully with content policies and contains no illegal or protected material. All data handled is legal and publicly accessible.