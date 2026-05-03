Find Top-Performing Instagram Reels & Save Insights to Notion with Gemini & Apify

https://n8nworkflows.xyz/workflows/find-top-performing-instagram-reels---save-insights-to-notion-with-gemini---apify-5795


# Find Top-Performing Instagram Reels & Save Insights to Notion with Gemini & Apify

### 1. Workflow Overview

This workflow, titled **"Find Top-Performing Instagram Reels & Save Insights to Notion with Gemini & Apify"**, is designed to automatically track trending Instagram Reels from selected sources, analyze their video content with Google Gemini (PaLM API), and store detailed insights in Notion databases. It targets social media analysts, content marketers, and digital teams wanting to monitor Instagram content performance and extract actionable data such as transcriptions, hooks, categories, and translations.

The workflow can be logically divided into the following blocks:

- **1.1 Trigger & Initialization:** Starts manually or on a schedule, sets variables and fetches Instagram source accounts from Notion.
- **1.2 Scraping & Data Retrieval:** Prepares and runs an Apify Instagram scraper actor, polls for completion, and retrieves scraped Reels data.
- **1.3 Data Mapping & Notion Synchronization:** Maps scraped Reels to Notion account pages, determines if Reels are new or existing, and updates or creates pages accordingly.
- **1.4 Video Content Analysis:** Downloads video, uploads it to Google Gemini API, polls for processing, sends analysis prompts, and formats AI response.
- **1.5 Final Data Storage & Looping:** Saves the enriched data back into Notion Reels database and updates account statuses.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger & Initialization

- **Overview:**  
  This block starts the workflow either manually or via a scheduled trigger, initializes essential variables, and fetches Instagram source accounts from the Notion "Sources" database.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Schedule Trigger  
  - Variables (Code)  
  - Get Sources (Notion)  

- **Node Details:**

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Allows manual start of the workflow for testing or ad-hoc runs.  
    - Configuration: Default, no parameters.  
    - Input/Output: No input; outputs trigger the Variables node.  
    - Edge Cases: None specific; manual trigger may be forgotten or missed if relying on schedule.

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Automatically triggers the workflow daily at 4 AM.  
    - Configuration: Set to trigger at hour 4 daily.  
    - Input/Output: No input; outputs trigger downstream nodes.  
    - Edge Cases: Timezone considerations; ensure n8n server time matches expected schedule.

  - **Variables**  
    - Type: Code  
    - Role: Defines configuration constants used throughout the workflow (e.g., Apify actor ID, days limits, translation language).  
    - Configuration: JavaScript object containing parameters like scrapingActorId, daysLimit, resultsLimit, maxDays, translationLang.  
    - Key Expressions: Hardcoded values for scraping actor and limits.  
    - Input/Output: Receives trigger, outputs JSON with variables.  
    - Edge Cases: Parameters must be updated for scale; if misconfigured, scraping or translation may fail.

  - **Get Sources**  
    - Type: Notion  
    - Role: Fetches all Instagram account pages from the "Sources" Notion database to scrape.  
    - Configuration: Operation 'getAll' on database named "Sources" with returnAll enabled.  
    - Credentials: Notion API integration required.  
    - Input/Output: Input from Variables node; outputs array of source accounts to Apify Payload node.  
    - Edge Cases: Notion API rate limits; database structure must match exactly; missing or malformed sources cause empty or incorrect scraping.

---

#### 1.2 Scraping & Data Retrieval

- **Overview:**  
  Builds the Apify scraper payload with Instagram URLs, runs the Apify Instagram scraper actor, checks the actor run status, implements wait logic, and retrieves the scraped Reels dataset.

- **Nodes Involved:**  
  - Apify Payload (Code)  
  - Run an Actor (Apify)  
  - Get Status (Apify)  
  - Switch (to handle actor run status)  
  - Wait (delay before polling again)  
  - Get dataset items (Apify)  

- **Node Details:**

  - **Apify Payload**  
    - Type: Code  
    - Role: Constructs the query payload for the Apify Instagram scraper actor using usernames from Notion sources.  
    - Configuration: Builds URLs array from usernames, sets resultsLimit, onlyPostsNewerThan (daysLimit), resultsType "stories".  
    - Key Expressions: Uses data from "Variables" and "Get Sources".  
    - Input/Output: Input from Get Sources; output to Run an Actor.  
    - Edge Cases: Empty usernames result in empty URLs; malformed URLs cause scraping failure.

  - **Run an Actor**  
    - Type: Apify  
    - Role: Executes the Instagram scraper actor with the prepared payload.  
    - Configuration: Uses scrapingActorId from Variables; passes the JSON query as customBody.  
    - Credentials: Requires Apify API key.  
    - Input/Output: Input from Apify Payload; outputs run ID to Get Status.  
    - Edge Cases: API rate limits, actor failures, malformed payload.

  - **Get Status**  
    - Type: Apify  
    - Role: Checks the status of the running Apify actor run.  
    - Configuration: Uses runId from "Run an Actor".  
    - Credentials: Apify API key.  
    - Input/Output: Input from Run an Actor and Wait nodes; outputs status to Switch.  
    - Edge Cases: Network issues, API timeouts, invalid runId.

  - **Switch**  
    - Type: Switch  
    - Role: Branches logic depending on actor run status: "SUCCEEDED", "RUNNING", or "READY".  
    - Configuration: Compares status string from Get Status.  
    - Input/Output: Input from Get Status; outputs to Get dataset items if "Done", or Wait if "Running" or "Ready".  
    - Edge Cases: Unknown status values; potential stuck states.

  - **Wait**  
    - Type: Wait  
    - Role: Introduces delay (1 minute) before polling Get Status again.  
    - Configuration: 1 minute wait.  
    - Input/Output: Input from Switch; outputs to Get Status.  
    - Edge Cases: Long-running actors might require longer wait; too short could cause rate limiting.

  - **Get dataset items**  
    - Type: Apify  
    - Role: Retrieves the scraped Instagram Reels data from Apify dataset after actor run completion.  
    - Configuration: DatasetId taken dynamically from actor run output; no limit set (all items).  
    - Credentials: Apify API key.  
    - Input/Output: Input from Switch "Done" branch; outputs to Map Reels.  
    - Edge Cases: Dataset missing or empty if actor failed; API errors.

---

#### 1.3 Data Mapping & Notion Synchronization

- **Overview:**  
  Processes scraped Reels to map them to Notion accounts, determines account activity status, compares with existing Notion Reels pages, and updates or creates Notion pages accordingly.

- **Nodes Involved:**  
  - Map Reels (Code)  
  - Owner? (If)  
  - Update Accounts (Notion)  
  - Many to One (Code)  
  - Get Reels (Notion)  
  - Code (for merging)  
  - Is Created? (Switch)  
  - Loop Over Items (SplitInBatches)  
  - Update (Notion)  
  - Create (Notion)  
  - Stats (Code)  

- **Node Details:**

  - **Map Reels**  
    - Type: Code  
    - Role: Assigns each Reel an activity status ("Active" or "Sleeping") based on how recent the Reel is relative to maxDays, and attaches Notion page IDs.  
    - Configuration: Uses Variables for maxDays; processes timestamps to determine status.  
    - Input/Output: Input from Get dataset items; outputs enriched items to Owner?.  
    - Edge Cases: Missing timestamps cause inaccurate status; map must be accurate to Notion IDs.

  - **Owner?**  
    - Type: If  
    - Role: Checks if the Reel item has an associated Notion account page ID.  
    - Configuration: Condition checks if notionPageId is not empty.  
    - Input/Output: Input from Map Reels; outputs to Update Accounts if true.  
    - Edge Cases: Missing notionPageId leads to skipping update.

  - **Update Accounts**  
    - Type: Notion  
    - Role: Updates Notion "Sources" pages with new status, title, and URL for each account.  
    - Configuration: Uses notionPageId for page to update; updates select status and URL properties.  
    - Credentials: Notion API.  
    - Input/Output: Input from Owner?; outputs to Many to One.  
    - Edge Cases: Notion API rate limits; missing pages cause failures.

  - **Many to One**  
    - Type: Code  
    - Role: Constructs a map from usernames to Notion page IDs to facilitate lookup.  
    - Configuration: Iterates over Notion accounts to build a dictionary.  
    - Input/Output: Input from Update Accounts; outputs to Get Reels.  
    - Edge Cases: Missing usernames cause incomplete mapping.

  - **Get Reels**  
    - Type: Notion  
    - Role: Retrieves all recent Reels from Notion "Reels" database filtered by date (after daysLimit).  
    - Configuration: Filter condition uses dynamic date calculation (daysLimit from Variables).  
    - Credentials: Notion API.  
    - Input/Output: Input from Many to One; outputs to Code node for merging.  
    - Edge Cases: Empty datasets if no Reels; API rate limits.

  - **Code** (Merge Reels)  
    - Type: Code  
    - Role: Merges scraped Reels with existing Notion Reels, deciding which need creation or update.  
    - Configuration: Compares URLs; assigns flags IsCreated and adds multiple analytics fields (Views, Likes, Comments, Saves, Shares).  
    - Input/Output: Inputs from Get Reels and Map Reels; outputs array of updates/creates to Loop Over Items.  
    - Edge Cases: URL mismatches cause duplicates; missing data fields.

  - **Loop Over Items**  
    - Type: SplitInBatches  
    - Role: Processes Reels one by one to allow sequential Notion create or update operations.  
    - Configuration: Default batch size (1).  
    - Input/Output: Input from Code node; outputs to Stats and Is Created? nodes.  
    - Edge Cases: Large datasets slow processing; batch size may need tuning.

  - **Stats**  
    - Type: Code  
    - Role: Dummy node adding a placeholder field; possibly for debugging or extension.  
    - Configuration: Adds a field 'myNewField' = 1 to each item.  
    - Input/Output: Input from Loop Over Items; outputs to Is Created?.  
    - Edge Cases: None significant.

  - **Is Created?**  
    - Type: Switch  
    - Role: Determines if the current Reel needs to be created new in Notion or updated based on IsCreated flag.  
    - Configuration: Checks boolean IsCreated and existence of notionAccountPageId.  
    - Input/Output: Input from Stats; outputs to Create or Update nodes.  
    - Edge Cases: Missing notionAccountPageId blocks both paths.

  - **Update**  
    - Type: Notion  
    - Role: Updates existing Notion Reels pages with new metrics (Likes, Comments, Views).  
    - Configuration: Uses notionPageId to specify page; updates numeric fields.  
    - Credentials: Notion API.  
    - Input/Output: Input from Is Created? "Update" branch; outputs back to Loop Over Items for next batch.  
    - Edge Cases: API limits, missing pages.

  - **Create**  
    - Type: Notion  
    - Role: Creates new pages in the Notion Reels database with enriched data including transcription, translation, and categorization.  
    - Configuration: Sets multiple properties including Date, Caption, Duration, Likes, URL, Views, Comments, Author relation, Content, Translation, Type, Category.  
    - Credentials: Notion API.  
    - Input/Output: Input from Format Response node; outputs to Loop Over Items.  
    - Edge Cases: Invalid property types, long text truncation, API limits, partial failures (configured to continue on error).

---

#### 1.4 Video Content Analysis

- **Overview:**  
  Downloads the Reels video, uploads it to Google Gemini API for analysis, polls the file processing state until ready, sends a prompt to analyze the content, and extracts JSON-formatted transcription, hooks, category, format, and translation.

- **Nodes Involved:**  
  - Download Video (HTTP Request)  
  - Upload to Gemini (HTTP Request)  
  - Processing Delay (Wait)  
  - Get File State (HTTP Request)  
  - Is Uploaded And Active? (If)  
  - Set Prompt (Set)  
  - Gemini Analyze (HTTP Request)  
  - Format Response (Code)  

- **Node Details:**

  - **Download Video**  
    - Type: HTTP Request  
    - Role: Downloads the video binary from the URL provided in the Reel data.  
    - Configuration: Uses videoUrl property as the request URL; no authentication.  
    - Input/Output: Input from Is Created? node; outputs binary data to Upload to Gemini.  
    - Edge Cases: Video URL inaccessible, network errors, timeouts.

  - **Upload to Gemini**  
    - Type: HTTP Request  
    - Role: Uploads downloaded video binary to Gemini API file upload endpoint.  
    - Configuration: POST to `generativelanguage.googleapis.com/upload/v1beta/files`, sends binary data as body, uses predefined Google Gemini (PaLM) API credentials.  
    - Credentials: Google Gemini (PaLM) API key.  
    - Input/Output: Input from Download Video; outputs file upload response to Processing Delay.  
    - Edge Cases: Authentication errors, upload failures.

  - **Processing Delay**  
    - Type: Wait  
    - Role: Waits 1 minute before polling file processing status.  
    - Configuration: 1 minute wait.  
    - Input/Output: Input from Upload to Gemini and Is Uploaded And Active? negative branch; outputs to Get File State.  
    - Edge Cases: Insufficient wait time causing premature status checks.

  - **Get File State**  
    - Type: HTTP Request  
    - Role: Polls Gemini API for current status of uploaded file using file URI.  
    - Configuration: GET request to file URI from upload response, with Google Gemini API credentials.  
    - Credentials: Google Gemini (PaLM) API key.  
    - Input/Output: Input from Processing Delay; outputs status to Is Uploaded And Active? node.  
    - Edge Cases: Network errors, rate limits.

  - **Is Uploaded And Active?**  
    - Type: If  
    - Role: Checks if the uploaded file is in "ACTIVE" state to proceed with analysis.  
    - Configuration: Condition equals `state == "ACTIVE"`.  
    - Input/Output: Input from Get File State; if yes, proceeds to Set Prompt; if no, loops back to Processing Delay.  
    - Edge Cases: File stuck in non-active states, causing infinite loop.

  - **Set Prompt**  
    - Type: Set  
    - Role: Constructs the AI prompt instructing Gemini to transcribe, find hooks, categorize, identify format, and translate the video content.  
    - Configuration: Hardcoded multi-step instructions with embedded variables for translation language from Variables node.  
    - Input/Output: Input from Is Uploaded And Active?; outputs JSON body for Gemini Analyze.  
    - Edge Cases: Prompt formatting errors, translationLang missing.

  - **Gemini Analyze**  
    - Type: HTTP Request  
    - Role: Sends prompt and file data to Gemini model "gemini-2.5-flash" for content generation.  
    - Configuration: POST to Gemini generateContent endpoint, JSON body includes file URI and text prompt, uses Google Gemini API credentials.  
    - Credentials: Google Gemini (PaLM) API key.  
    - Input/Output: Input from Set Prompt; outputs AI response JSON to Format Response.  
    - Edge Cases: API errors, malformed requests, rate limits.

  - **Format Response**  
    - Type: Code  
    - Role: Cleans and parses Gemini's JSON output, merges it into the Reel data with fields like transcription, hook, category, format, translation, and translation_hook.  
    - Configuration: Strips markdown code fences from response, parses JSON, adds defaults if missing.  
    - Input/Output: Input from Gemini Analyze; outputs enriched Reel data to Create node.  
    - Edge Cases: Invalid JSON causing parse errors, incomplete AI output.

---

#### 1.5 Final Data Storage & Looping

- **Overview:**  
  After processing, the workflow loops back to handle all Reels sequentially, updating or creating Notion pages and refreshing account mappings, enabling continuous monitoring.

- **Nodes Involved:**  
  - Loop Over Items (SplitInBatches)  
  - Stats (Code)  
  - Is Created? (Switch)  
  - Update (Notion)  
  - Create (Notion)  
  - Update Accounts (Notion)  
  - Many to One (Code)  
  - Get Reels (Notion)  
  - Code (merge Reels)  

- **Node Details:**  
  - This block is effectively integrated with 1.3; it ensures each Reel is processed individually with proper Notion updates and creates, and maintains mapping consistency.  
  - Edge cases include API limits, batch processing delays, and error continuation on create failures.

---

### 3. Summary Table

| Node Name                 | Node Type                 | Functional Role                            | Input Node(s)                  | Output Node(s)               | Sticky Note                                                                                                                         |
|---------------------------|---------------------------|--------------------------------------------|-------------------------------|------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger            | Manual start of workflow                     | -                             | Variables                    |                                                                                                                                     |
| Schedule Trigger          | Schedule Trigger          | Scheduled daily trigger                       | -                             | Variables                    |                                                                                                                                     |
| Variables                | Code                      | Defines config variables                      | When clicking ‘Execute workflow’, Schedule Trigger | Get Sources                 |                                                                                                                                     |
| Get Sources              | Notion                    | Fetches Instagram source accounts            | Variables                     | Apify Payload                | Database = Sources                                                                                                                   |
| Apify Payload            | Code                      | Builds Apify scraper query                    | Get Sources                   | Run an Actor                 |                                                                                                                                     |
| Run an Actor             | Apify                     | Runs Instagram scraper actor                  | Apify Payload                 | Get Status                   |                                                                                                                                     |
| Get Status               | Apify                     | Checks actor run status                        | Run an Actor, Wait            | Switch                      |                                                                                                                                     |
| Switch                   | Switch                    | Branches on actor status                       | Get Status                    | Get dataset items, Wait      |                                                                                                                                     |
| Wait                     | Wait                      | Delays polling for actor completion           | Switch                       | Get Status                   |                                                                                                                                     |
| Get dataset items        | Apify                     | Retrieves scraped Instagram Reels             | Switch                       | Map Reels                   |                                                                                                                                     |
| Map Reels                | Code                      | Assigns activity status and Notion IDs        | Get dataset items             | Owner?                      |                                                                                                                                     |
| Owner?                   | If                        | Checks if Reel linked to Notion account       | Map Reels                    | Update Accounts             |                                                                                                                                     |
| Update Accounts          | Notion                    | Updates Notion source account pages           | Owner?                       | Many to One                 |                                                                                                                                     |
| Many to One              | Code                      | Creates username-to-Notion page ID mapping    | Update Accounts              | Get Reels                   |                                                                                                                                     |
| Get Reels                | Notion                    | Fetches recent Reels from Notion database     | Many to One                  | Code (merge)                | Database = Reels                                                                                                                    |
| Code (merge Reels)       | Code                      | Merges scraped and existing Reels data        | Get Reels, Map Reels          | Loop Over Items             |                                                                                                                                     |
| Loop Over Items          | SplitInBatches            | Processes Reels one by one                     | Code (merge Reels)            | Stats, Is Created?          |                                                                                                                                     |
| Stats                    | Code                      | Adds placeholder field (debug/extension)      | Loop Over Items               | Is Created?                 |                                                                                                                                     |
| Is Created?              | Switch                    | Decides create or update Notion page          | Stats                        | Create, Update              |                                                                                                                                     |
| Create                   | Notion                    | Creates new Reel pages in Notion               | Format Response               | Loop Over Items             | Database = Reels                                                                                                                    |
| Update                   | Notion                    | Updates existing Reel pages in Notion          | Is Created?                  | Loop Over Items             |                                                                                                                                     |
| Download Video           | HTTP Request              | Downloads Reel video binary                     | Is Created? "Create" branch   | Upload to Gemini            |                                                                                                                                     |
| Upload to Gemini         | HTTP Request              | Uploads video to Gemini API                      | Download Video               | Processing Delay            |                                                                                                                                     |
| Processing Delay         | Wait                      | Waits before polling Gemini file status         | Upload to Gemini, Is Uploaded And Active? (no) | Get File State              |                                                                                                                                     |
| Get File State           | HTTP Request              | Polls Gemini API for file processing status     | Processing Delay             | Is Uploaded And Active?      |                                                                                                                                     |
| Is Uploaded And Active?  | If                        | Checks if uploaded file is ACTIVE                | Get File State               | Set Prompt, Processing Delay |                                                                                                                                     |
| Set Prompt               | Set                       | Creates prompt for Gemini content analysis       | Is Uploaded And Active?       | Gemini Analyze              | This is the prompt used to process video content and classify its category and type. Customize it based on your logic.             |
| Gemini Analyze           | HTTP Request              | Sends prompt and file to Gemini for analysis     | Set Prompt                   | Format Response             |                                                                                                                                     |
| Format Response          | Code                      | Parses Gemini response and enriches Reel data    | Gemini Analyze               | Create                     |                                                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Notion Databases:**  
   - "Sources": Holds Instagram accounts with columns including `property_username`, `Status|select`, `Title|title`, `URL|url`.  
   - "Reels": Holds Reel data with columns such as `Date|date`, `Caption|rich_text`, `Duration|number`, `Likes|number`, `Views|number`, `Comments|number`, `Author|relation` (linked to Sources), `Content|rich_text`, `Translation|rich_text`, `Type|select`, `Category|select`.

2. **Create Credentials:**  
   - Notion API credential with access to the above databases.  
   - Apify API credential with access to your Apify account.  
   - Google Gemini (PaLM) API credential with your API key (used in HTTP Request nodes).

3. **Create Trigger Nodes:**  
   - Manual Trigger node named "When clicking ‘Execute workflow’" (no configuration).  
   - Schedule Trigger node named "Schedule Trigger" set to trigger daily at 4 AM.

4. **Create Variables Node:**  
   - Code node named "Variables" with JS code defining:  
     ```
     {
       scrapingActorId: "shu8hvrXbJbY3Eb9W",
       daysLimit: 7,
       resultsLimit: 3,
       maxDays: 2,
       translationLang: "German"
     }
     ```

5. **Get Instagram Sources from Notion:**  
   - Notion node named "Get Sources", operation "getAll", resource "databasePage", databaseId linked to "Sources". Return all pages.

6. **Build Apify Payload:**  
   - Code node "Apify Payload" that builds an Apify scraper query from usernames in Sources:  
     - Fields: directUrls (Instagram profile URLs), resultsLimit, onlyPostsNewerThan (e.g., "7 days"), resultsType = "stories".

7. **Run Apify Actor:**  
   - Apify node "Run an Actor":  
     - actorId from Variables.scrapingActorId  
     - operation: Run actor  
     - customBody: JSON query from Apify Payload

8. **Poll Actor Status:**  
   - Apify node "Get Status" with runId from "Run an Actor".  
   - Switch node "Switch" to branch on status:  
     - "SUCCEEDED" → continue to get dataset items.  
     - "RUNNING" or "READY" → Wait node (1 minute) then loop back to "Get Status".

9. **Get Scraped Dataset:**  
   - Apify node "Get dataset items" with datasetId from actor run result.

10. **Map Reels Data:**  
    - Code node "Map Reels" that assigns activity status ("Active"/"Sleeping") based on dates, attaches Notion page IDs from Apify payload map.

11. **Update Account Pages in Notion:**  
    - If node "Owner?" checks if reel has notionPageId.  
    - Notion node "Update Accounts" updates account status and details.  
    - Code node "Many to One" creates map username → Notion page ID.

12. **Fetch Existing Reels:**  
    - Notion node "Get Reels" gets Reels from Notion filtered by date (after daysLimit).

13. **Merge and Decide Create/Update:**  
    - Code node merges existing with scraped Reels, marks new versus existing, prepares update properties.

14. **Process Reels in Batches:**  
    - SplitInBatches node "Loop Over Items" to process each Reel individually.

15. **Add Debug Stats (optional):**  
    - Code node "Stats" adds dummy field.

16. **Switch Create or Update:**  
    - Switch node "Is Created?" branches to Notion "Create" or "Update".

17. **Update Existing Reel Pages:**  
    - Notion "Update" node updates Likes, Views, Comments.

18. **Create New Reel Pages:**  
    - Notion "Create" node creates pages with all enriched fields including transcription, translation, categories.

19. **Download Video:**  
    - HTTP Request node "Download Video" downloads Reel video by URL.

20. **Upload Video to Gemini:**  
    - HTTP Request node "Upload to Gemini" uploads video binary to Gemini API endpoint.

21. **Wait for Processing:**  
    - Wait node "Processing Delay" waits 1 minute before checking file status.

22. **Poll File Status:**  
    - HTTP Request node "Get File State" to check upload status.

23. **Check if File Ready:**  
    - If node "Is Uploaded And Active?" checks if file state is "ACTIVE".

24. **Set Gemini Prompt:**  
    - Set node "Set Prompt" builds the JSON prompt for transcription, hook extraction, categorization, format detection, and translation using Variables.translationLang.

25. **Analyze Video Content:**  
    - HTTP Request node "Gemini Analyze" sends prompt and file to Gemini model for content generation.

26. **Format Gemini Response:**  
    - Code node "Format Response" parses Gemini output JSON, merges fields into Reel data.

27. **Loop Back:**  
    - Newly created or updated Reel data flows back to batch processing for further updates.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                               | Context or Link                                                                                                                     |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------|
| ### 📺 How It Works – Watch the Video  \nI've recorded a video walkthrough showing system details:  \n👉 https://www.youtube.com/watch?v=rdfRNHpHX8o                                                                                         | Video walkthrough explaining the workflow                                                                                          |
| ### 📄 Download Notion Database Structure  \nDownload Notion table structure with columns and formats:  \n👉 https://drive.google.com/file/d/1FVaS_-ztp6PDAJbETUb1dkg8IqE4qHqp/view?usp=sharing                                                | Notion database template for Sources and Reels                                                                                     |
| ### ☕ Support the Project  \nA version with tips is here:  \n👉 https://gr.egrnkvch.com/l/InstagramReelsTrendWatcher                                                                                                                     | Support and tips version link                                                                                                      |
| ## ⚙️ How to Install the Template  \n1. Create Notion databases with exact columns and formats.  \n2. Import workflow template into n8n.  \n3. Add Notion and Apify credentials.  \n4. Use Google Gemini API with HTTP Request nodes and proper credentials.  \n5. Configure Variables node parameters. | Installation instructions sticky note                                                                                            |
| ### 📺 Video Guide  \n👉 https://www.youtube.com/watch?v=rdfRNHpHX8o                                                                                                                                                                       | Additional video guide link                                                                                                        |
| Database = Sources                                                                                                                                                                                                                        | Sticky note near Get Sources node                                                                                                  |
| Database = Reels                                                                                                                                                                                                                          | Sticky notes near Get Reels and Create nodes                                                                                       |
| This is the prompt used to process video content and classify its category and type. Customize it based on your logic.                                                                                                                    | Sticky note near Set Prompt node                                                                                                   |

---

This comprehensive documentation describes the full logic, nodes, and configurations of the "Reels Trends Watcher" workflow, enabling advanced users or AI agents to understand, reproduce, and modify it effectively.