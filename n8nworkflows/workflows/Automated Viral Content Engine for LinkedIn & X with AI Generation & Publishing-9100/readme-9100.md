Automated Viral Content Engine for LinkedIn & X with AI Generation & Publishing

https://n8nworkflows.xyz/workflows/automated-viral-content-engine-for-linkedin---x-with-ai-generation---publishing-9100


# Automated Viral Content Engine for LinkedIn & X with AI Generation & Publishing

### 1. Workflow Overview

**Title:** Automated Viral Content Engine for LinkedIn & X with AI Generation & Publishing

**Purpose:**  
This workflow automates the entire lifecycle of viral content creation and publishing for LinkedIn and X (formerly Twitter). It discovers trending posts in a niche (AI and automation), analyzes and classifies them, generates new posts in a tailored style, creates accompanying images, repurposes content for different platforms, and finally publishes the content automatically. The system also supports manual triggers and on-demand commands via Telegram.

**Target Use Cases:**  
- Social media managers or content creators targeting AI and automation niches.  
- Entrepreneurs or marketers seeking to automate content discovery, creation, and publishing.  
- Users wanting multi-platform content repurposing (LinkedIn and X).  
- Leveraging AI agents to generate engaging posts based on viral content.  
- Automating social media publishing with human-in-the-loop validation.

**Logical Blocks:**  
- **1.1 Initialization and Competitor Tracking:** Collect competitor profiles and update a database for content monitoring.  
- **2. Discovery of Viral Content:** Scrape viral LinkedIn posts from competitors and classify their content into evergreen or personal categories.  
- **3. Content Generation and Post Creation:** AI-based rewriting and image generation for viral posts to produce ready-to-publish LinkedIn content.  
- **4. Repurposing for X (Twitter):** Convert LinkedIn posts into X-friendly format (tweets and threads).  
- **5. Automated Publishing:** Scheduled posting to LinkedIn and X with media attachments, including error handling and status updates.  
- **6. Telegram Integration and On-Demand Commands:** Manage workflow execution, content scraping, and AI-driven post generation triggered by Telegram commands.  
- **7. Research and Verification:** Optional external research to augment content credibility and details.

---

### 2. Block-by-Block Analysis

#### 1.1 Initialization and Competitor Tracking

**Overview:**  
This optional block builds and maintains a list of competitor LinkedIn profiles. It scrapes posts from these profiles and filters for engagement to identify relevant profiles to monitor.

**Nodes Involved:**  
- Manual Trigger (disabled)  
- Edit Fields  
- Find Competitors (HTTP Request to Apify)  
- Add Competitors (Google Sheets)  
- >100 likes (Filter)  
- Get Competitors (Google Sheets)  
- Aggregate  

**Node Details:**  

- **Manual Trigger**  
  - Type: Manual trigger to start the workflow. Disabled by default.  
  - Purpose: Manual initiation of competitor scraping.  
  - Connections: None (start node).  
  - Notes: Disabled; scheduled or Telegram trigger preferred.

- **Edit Fields**  
  - Type: Set node to format data fields.  
  - Configuration: Extracts and maps author name, LinkedIn URL, and headline from incoming JSON.  
  - Inputs: Posts from competitor scraping.  
  - Outputs: Formatted competitor data.  
  - Risks: Expression errors if expected fields missing.

- **Find Competitors**  
  - Type: HTTP Request to Apify API for LinkedIn scraping.  
  - Configuration: Scrapes posts based on predefined keyword queries related to AI and automation. Queries include “ai automation”, “n8n automation”, etc.  
  - Authentication: Apify API key (generic HTTP header auth).  
  - Outputs: JSON with posts and author info.  
  - Risks: API rate limits, network errors, malformed responses.

- **>100 likes**  
  - Type: Filter node.  
  - Configuration: Only passes posts with more than 100 likes.  
  - Risks: Missing engagement data can cause filter failure.

- **Add Competitors**  
  - Type: Google Sheets append/update.  
  - Configuration: Records competitor’s name, LinkedIn URL, and headline into “Competitors” sheet, updating existing entries by LinkedIn URL.  
  - Credentials: Google Sheets OAuth (configured).  
  - Risks: Google API quota, credential expiry.

- **Get Competitors**  
  - Type: Google Sheets read operation.  
  - Configuration: Reads competitor profiles from “Competitors” sheet for use in scraping viral posts.  
  - Credentials: Google Sheets OAuth.  
  - Risks: Sheet access issues.

- **Aggregate**  
  - Type: Aggregate node.  
  - Configuration: Aggregates LinkedIn URLs from competitor profiles into an array for downstream use.  
  - Risks: Empty input causing empty aggregation.

---

#### 2. Discovery of Viral Content

**Overview:**  
This block fetches viral posts from LinkedIn profiles of competitors, filters for posts with high engagement, removes duplicates, classifies the posts into evergreen or personal categories, and appends the evergreen ideas for further processing.

**Nodes Involved:**  
- Saturday 11 PM (Schedule Trigger)  
- Get Competitors (Google Sheets)  
- Scrape Viral Posts (HTTP Request to Apify)  
- >100 likes (Filter)  
- Remove Duplicates  
- Text Classifier (LangChain)  
- Append Ideas (Google Sheets)  
- Notify (Telegram)  

**Node Details:**  

- **Saturday 11 PM**  
  - Type: Scheduled trigger at Saturday 11 PM weekly.  
  - Purpose: Automate weekly scraping of viral posts.

- **Scrape Viral Posts**  
  - Type: HTTP Request to Apify API for post scraping.  
  - Inputs: List of competitor LinkedIn URLs.  
  - Configuration: Scrapes 10 posts with max 5 comments and 5 reactions each, limited to recent posts within last week.  
  - Risks: API limits, invalid URLs.

- **Remove Duplicates**  
  - Type: Remove duplicate posts by unique identifier.  
  - Purpose: Prevent processing same posts multiple times.

- **Text Classifier**  
  - Type: LangChain text classification node using a custom prompt.  
  - Configuration: Classifies posts into “EvergreenAI” or “Personal/Other” based on content quality and relevance to AI automation.  
  - Risks: Classification errors, timeouts, prompt misconfiguration.

- **Append Ideas**  
  - Type: Google Sheets append operation.  
  - Configuration: Stores evergreen posts along with metadata into “Viral Ideas” sheet for later use.  
  - Risks: Google API quota.

- **Notify (Telegram)**  
  - Type: Telegram message node.  
  - Configuration: Sends notification with link to “Viral Ideas” sheet and instructions for human review.

---

#### 3. Content Generation and Post Creation

**Overview:**  
This block generates rewritten LinkedIn posts in the user’s style based on viral content, creates minimalistic images for posts, and stores the generated content and images for review and publishing.

**Nodes Involved:**  
- Sunday 1 AM (Schedule Trigger)  
- Get Viral Ideas (Google Sheets)  
- Loop Over Items (Split batch processing)  
- Post Repurpose (LangChain Agent)  
- Research (Sub-workflow for fact-checking)  
- Gemini Flash (LangChain Google Gemini for image generation)  
- HTTP Request (Google Gemini image generation API)  
- Minimalistic (Google Drive file download - style reference)  
- Convert to File (binary conversion of image)  
- Save Image (Upload to Google Drive)  
- Save Post (Append to “Content Gen” Google Sheet)  
- Mark Idea as Done (Update “Viral Ideas” row status)  
- Notify (Telegram)  

**Node Details:**  

- **Get Viral Ideas**  
  - Reads posts marked as “Ready” for generation.

- **Loop Over Items**  
  - Processes each post individually to maintain flow control.

- **Post Repurpose**  
  - AI agent rewrites viral post using custom prompt with detailed instructions on tone, style, and content buckets.  
  - Includes instructions to add CTAs, maintain factual accuracy, and generate a minimalistic image description.  
  - Risks: AI hallucination, prompt misinterpretation, API quotas.

- **Research**  
  - Sub-workflow triggered when additional external info is required for fact-based posts.  
  - Uses a web scraping agent (browseract) to retrieve supplemental data.  
  - Risks: Scraper failures, incomplete data.

- **Gemini Flash & HTTP Request**  
  - Generate post images based on AI-generated descriptions and style reference image downloaded from Google Drive.  
  - Risks: API failures, rate limits, image generation errors.

- **Minimalistic**  
  - Downloads a style reference image from Google Drive to guide image generation.

- **Convert to File**  
  - Converts base64 image data into binary for upload.

- **Save Image**  
  - Uploads generated image to Google Drive folder for storage and access.

- **Save Post**  
  - Stores generated post text and image URLs into “Content Gen” sheet with status “Waiting”.

- **Mark Idea as Done**  
  - Updates original viral idea entry to “All Done” to prevent reprocessing.

- **Notify**  
  - Sends Telegram notification to user with link to generated content for review.

---

#### 4. Repurposing for X (Twitter)

**Overview:**  
Generates X (Twitter) posts and threads from LinkedIn content, formats them for platform constraints, and queues them for publishing.

**Nodes Involved:**  
- Sunday 7 AM (Schedule Trigger)  
- Get Content (Google Sheets - LinkedIn posts)  
- Loop Over Items (Batch processing)  
- Content Repurpose (LangChain LLM chain for tweet generation)  
- Content Repurpose 1 (LangChain for thread generation)  
- Structured Output Parser (JSON parsing)  
- Append (Google Sheets - “Other handles”)  
- Mark Idea as Done (Update LinkedIn post status)  
- Notify (Telegram)  

**Node Details:**  

- **Get Content**  
  - Retrieves LinkedIn posts with status “Posted” for repurposing.

- **Content Repurpose**  
  - Converts LinkedIn post to a concise, single tweet (≤280 characters) using LangChain with instructions to maintain tone and avoid banned language.

- **Content Repurpose 1**  
  - Converts LinkedIn post into multi-tweet thread with logical flow and best practices.

- **Structured Output Parser**  
  - Parses AI JSON response to extract tweets or threads.

- **Append**  
  - Stores generated tweet/thread content in “Other handles” sheet with status “Waiting”.

- **Mark Idea as Done**  
  - Marks LinkedIn post as “All Done” to prevent duplicate repurposing.

- **Notify**  
  - Sends Telegram notification with generated X content for review.

---

#### 5. Automated Publishing

**Overview:**  
Schedules automated posting of approved LinkedIn and X posts, handling media upload and status updates, with error handling and notifications.

**Nodes Involved:**  
- 6 PM (Schedule Trigger for LinkedIn)  
- 6:30 PM (Schedule Trigger for X)  
- Get Content (Google Sheets)  
- Get File (Google Drive - image download)  
- Image Valid? (Conditional check)  
- Upload Media (X via OAuth 1.0a)  
- Create Post (LinkedIn API)  
- Create Tweet (X API)  
- Update Status (Google Sheets)  
- Telegram Notifications (Success or failure messages)  

**Node Details:**  

- **Get Content**  
  - Selects posts ready to be published.

- **Get File**  
  - Downloads post image from Google Drive for media upload.

- **Image Valid?**  
  - Checks if image exists and is marked valid.

- **Upload Media**  
  - Uploads image to X via OAuth 1.0a media upload endpoint.

- **Create Post**  
  - Publishes post on LinkedIn with text and image.

- **Create Tweet**  
  - Posts tweet with or without image on X platform.

- **Update Status**  
  - Updates post status in Google Sheets to “Posted” and records posting date.

- **Telegram Nodes**  
  - Notify user upon success or failure of publishing.

**Risks:**  
- API rate limits and failures (LinkedIn, X, Google).  
- OAuth credentials expiration or misconfiguration.  
- Media upload failures leading to fallback to text-only posts.

---

#### 6. Telegram Integration and On-Demand Commands

**Overview:**  
Provides Telegram bot interface for manual workflow triggers, user commands for scraping, post generation, and content repurposing with access control.

**Nodes Involved:**  
- Telegram Trigger  
- Switch (Command router)  
- Notify Start (Telegram)  
- Execute Sub-Workflows (Scrape viral posts, Generate LinkedIn post, Generate X post)  
- Error Handling (Telegram messages for invalid users or commands)  

**Node Details:**  

- **Telegram Trigger**  
  - Listens to messages and filters by user chat ID to restrict access.

- **Switch**  
  - Routes commands based on message content:  
    - `/sms.scrape` triggers viral content scraping.  
    - `/sms.linkedin:` triggers LinkedIn post generation.  
    - `/sms.x:` triggers X post generation.  
    - Other inputs routed to error messages.

- **Execute Sub-Workflows**  
  - Launches respective workflows asynchronously without waiting.

- **Notify Start**  
  - Sends confirmation message upon workflow start.

- **Error Handling**  
  - Sends error message for unauthorized users or invalid commands.

**Risks:**  
- Unauthorized access if user ID not set properly.  
- Unhandled commands causing user confusion.  
- Telegram API limits or downtime.

---

#### 7. Research and Verification

**Overview:**  
Supports content fact-checking and enrichment via external web scraping tools integrated as sub-workflows.

**Nodes Involved:**  
- Research (Sub-workflow invoking browseract agent)  
- Perplexity (disabled, alternative paid research)  
- Output nodes parsing research data  

**Node Details:**  

- **Research**  
  - Invokes external scraping agent with user query for content verification or data enrichment.

- **Perplexity** (disabled)  
  - Calls paid AI search API for advanced content research.

- **Output Handling**  
  - Parses and integrates research results into AI-generated posts.

**Risks:**  
- Web scraper reliability and speed.  
- API failures causing incomplete data.  
- Cost implications for paid APIs.

---

### 3. Summary Table

| Node Name             | Type                                | Functional Role                                 | Input Node(s)             | Output Node(s)          | Sticky Note                                                    |
|-----------------------|-----------------------------------|------------------------------------------------|---------------------------|-------------------------|---------------------------------------------------------------|
| Manual Trigger        | Manual trigger                    | Optional manual start of competitor scraping   | None                      | Find Competitors         |                                                               |
| Edit Fields           | Set                              | Format competitor data fields                    | Find Competitors           | Add Competitors          |                                                               |
| Find Competitors      | HTTP Request                     | Scrape LinkedIn profiles matching keywords      | Manual Trigger             | >100 likes               |                                                               |
| Add Competitors       | Google Sheets                    | Store/update competitor info                      | Edit Fields                |                           |                                                               |
| >100 likes            | Filter                          | Filter posts with more than 100 likes            | Find Competitors           | Add Competitors          |                                                               |
| Get Competitors       | Google Sheets                    | Read competitor profiles                          | None                      | Scrape Viral Posts       |                                                               |
| Aggregate             | Aggregate                       | Aggregate LinkedIn URLs                           | Get Competitors            | Scrape Viral Posts       |                                                               |
| Saturday 11 PM        | Schedule Trigger                | Weekly trigger for viral post scraping           | None                      | Get Competitors           |                                                               |
| Scrape Viral Posts    | HTTP Request                   | Scrape viral posts from competitor profiles      | Aggregate                  | >100 likes               |                                                               |
| Remove Duplicates     | Remove Duplicates               | Remove duplicate posts                            | >100 likes                 | Text Classifier          |                                                               |
| Text Classifier       | LangChain Text Classifier       | Classify posts into evergreen or personal types | Remove Duplicates           | Append Ideas             |                                                               |
| Append Ideas          | Google Sheets                  | Append evergreen ideas to sheet                   | Text Classifier             | Notify                   |                                                               |
| Notify                | Telegram                       | Send notification about scraping                  | Append Ideas                | None                     |                                                               |
| Sunday 1 AM           | Schedule Trigger               | Trigger post generation                           | None                      | Get Viral Ideas           |                                                               |
| Get Viral Ideas       | Google Sheets                 | Fetch viral posts ready for processing            | None                      | Loop Over Items           |                                                               |
| Loop Over Items       | Split Batch                   | Process posts individually                         | Get Viral Ideas             | Post Repurpose            |                                                               |
| Post Repurpose        | LangChain Agent               | Rewrite posts in user tone & generate image desc | Loop Over Items             | Gemini Flash              |                                                               |
| Research              | Sub-workflow                  | Optional research for content enrichment          | Post Repurpose              | Post Repurpose            |                                                               |
| Gemini Flash          | LangChain Google Gemini       | Generate AI images for posts                       | Post Repurpose              | HTTP Request (Image Gen)  |                                                               |
| HTTP Request (Image)  | HTTP Request                 | Call Gemini image generation API                   | Gemini Flash                | Convert to File           |                                                               |
| Minimalistic          | Google Drive Download         | Download style reference image                     | None                      | HTTP Request (Image Gen)  |                                                               |
| Convert to File       | Convert to File               | Convert base64 to binary image                      | HTTP Request (Image Gen)   | Save Image                |                                                               |
| Save Image            | Google Drive Upload          | Upload generated image                              | Convert to File             | Save Post                 |                                                               |
| Save Post             | Google Sheets                | Store generated post and image info                 | Post Repurpose              | Mark Idea as Done         |                                                               |
| Mark Idea as Done     | Google Sheets                | Mark original idea as processed                     | Save Post                  | None                     |                                                               |
| Notify (Telegram)     | Telegram                     | Notify user with post generation status             | Save Post                  | None                     |                                                               |
| Sunday 7 AM           | Schedule Trigger             | Trigger repurposing for X (Twitter)                 | None                      | Get Content               |                                                               |
| Get Content           | Google Sheets                | Get LinkedIn posts for repurposing                   | None                      | Loop Over Items           |                                                               |
| Loop Over Items       | Split Batch                 | Process posts individually                           | Get Content                | Content Repurpose         |                                                               |
| Content Repurpose     | LangChain LLM Chain          | Generate X tweet from LinkedIn post                  | Loop Over Items             | Structured Output Parser  |                                                               |
| Content Repurpose 1   | LangChain LLM Chain          | Generate X tweet thread from LinkedIn post           | Loop Over Items             | Structured Output Parser  |                                                               |
| Structured Output Parser | LangChain Parser           | Parse AI output JSON                                  | Content Repurpose           | Append                    |                                                               |
| Append                | Google Sheets                | Append X posts and threads                            | Structured Output Parser    | Mark Idea as Done         |                                                               |
| Mark Idea as Done     | Google Sheets                | Mark LinkedIn post as done after repurposing          | Append                     | Notify                    |                                                               |
| Notify (Telegram)     | Telegram                     | Notify user about repurposed content ready            | Mark Idea as Done           | None                      |                                                               |
| 6 PM                  | Schedule Trigger             | Trigger LinkedIn auto-posting                          | None                      | Get Content (LinkedIn)    |                                                               |
| Get Content (LinkedIn)| Google Sheets                | Get LinkedIn posts with status “Waiting”               | None                      | Get File                  |                                                               |
| Get File              | Google Drive Download        | Download LinkedIn post image                            | Get Content                | Image Valid?              |                                                               |
| Image Valid?          | If                           | Check image validity                                    | Get File                   | Upload Media / Create Post|                                                               |
| Upload Media (X)      | HTTP Request                 | Upload image to X media endpoint                        | Image Valid? (true)         | Create Tweet              |                                                               |
| Create Post (LinkedIn)| LinkedIn API                | Publish post with image to LinkedIn                     | Image Valid? / Get File     | Update Status             |                                                               |
| Create Tweet (X)      | Twitter API                 | Publish tweet with or without image                      | Upload Media / Image Valid? | Update Status             |                                                               |
| Update Status         | Google Sheets                | Update post status to “Posted” and date                  | Create Post / Create Tweet  | Notify                    |                                                               |
| Notify (Telegram)     | Telegram                     | Notify user about successful posting                     | Update Status              | None                      |                                                               |
| Telegram Trigger      | Telegram                     | Listen to user commands                                  | None                      | Switch                    |                                                               |
| Switch                | Switch                       | Route Telegram commands                                  | Telegram Trigger           | Execute Sub-workflows     |                                                               |
| Execute Sub-Workflows | Execute Workflow             | Launch sub-workflows based on command                    | Switch                    | Notify                    |                                                               |
| Notify Start          | Telegram                     | Notify user of workflow start                             | Execute Sub-Workflows      | None                      |                                                               |
| Error Handling        | Telegram                     | Notify unauthorized users or invalid commands            | Switch                    | None                      |                                                               |
| Research (Sub-workflow) | Sub-workflow               | External data scraping for content enrichment             | Post Repurpose / Post Generator | Post Repurpose / Post Generator |                                                               |

---

### 4. Reproducing the Workflow from Scratch

**Prerequisites:**  
- A working n8n instance (latest recommended version).  
- Credentials: Google Sheets OAuth, Google Drive OAuth, Telegram Bot API, Apify API, OpenAI/Google Gemini API, X (Twitter) OAuth (both OAuth 1.0a and OAuth 2).  
- Google Sheets prepared with sheets: “Competitors”, “Viral Ideas”, “Content Gen”, “Other handles”.  
- Telegram Bot created with chat ID known.  
- Apify actors configured for LinkedIn scraping.  
- Browseract agent for research (optional).

---

**Step-by-Step Build:**

**1. Initialization and Competitor Tracking**  
1. Create a **Manual Trigger** node (disabled by default).  
2. Add a **HTTP Request** node (“Find Competitors”) configured to POST to Apify LinkedIn scraper with JSON body containing AI-related keywords. Authenticate with Apify API key.  
3. Add a **Filter** node (“>100 likes”) to allow only posts with likes > 100.  
4. Add a **Set** node (“Edit Fields”) to extract and map author name, LinkedIn URL, headline from scraped JSON.  
5. Add a **Google Sheets** node (“Add Competitors”) to append/update competitor info to the Competitors sheet. Use Google Sheets OAuth credentials.  
6. Add a **Google Sheets** node (“Get Competitors”) to read competitor profiles from Competitors sheet.  
7. Add an **Aggregate** node to collect LinkedIn URLs into a list. Connect this to the “Scrape Viral Posts” node in the next block.

---

**2. Discovery of Viral Content**  
1. Create a **Schedule Trigger** node (“Saturday 11 PM”) to run weekly.  
2. Connect to **Get Competitors** node (Google Sheets) to fetch competitor data.  
3. Add a **HTTP Request** node (“Scrape Viral Posts”) to call Apify API for scraping viral posts from competitor LinkedIn profiles. Use competitor URLs from aggregation.  
4. Filter posts with likes > 100 using Filter node.  
5. Add a **Remove Duplicates** node to eliminate duplicate posts by unique ID.  
6. Add a **LangChain Text Classifier** node with custom prompt to classify posts as “EvergreenAI” or “Personal/Other” based on content.  
7. Append classified evergreen posts to “Viral Ideas” sheet using Google Sheets node.  
8. Add a **Telegram** node to notify user with the link to the Viral Ideas sheet.

---

**3. Content Generation and Post Creation**  
1. Create a **Schedule Trigger** node (“Sunday 1 AM”).  
2. Add a **Google Sheets** node (“Get Viral Ideas”) to fetch posts with status “Ready”.  
3. Add a **Split Batch** node (“Loop Over Items”) to process each post separately.  
4. Add a **LangChain Agent** node (“Post Repurpose”) with a detailed prompt instructing AI to rewrite the post in user’s tone, create an intro, detailed content, and image description.  
5. Create a **Sub-workflow** (“Research”) invoked when research is needed for factual accuracy and add it here.  
6. Add a **LangChain Google Gemini** node (“Gemini Flash”) to generate a minimalistic image based on AI description and style reference.  
7. Use a **Google Drive** node (“Minimalistic”) to download the style reference image from a shared folder.  
8. Add an **HTTP Request** node to call Gemini’s image generation API with style and description.  
9. Add **Convert to File** node to convert image base64 data to binary format.  
10. Upload image to Google Drive folder with **Google Drive Upload** node (“Save Image”).  
11. Save post text, image URL, and metadata to “Content Gen” Google Sheet using **Google Sheets** node (“Save Post”).  
12. Mark processed viral idea as “All Done” in “Viral Ideas” sheet.  
13. Notify user with Telegram node of completion.

---

**4. Repurposing for X (Twitter)**  
1. Create a **Schedule Trigger** node (“Sunday 7 AM”).  
2. Use **Google Sheets** node (“Get Content”) to fetch LinkedIn posts with status “Posted”.  
3. Add **Split Batch** node (“Loop Over Items”) for processing.  
4. Add **LangChain LLM Chain** node (“Content Repurpose”) with prompt to generate concise single tweets (≤280 chars).  
5. Add another **LangChain LLM Chain** node (“Content Repurpose 1”) to generate tweet threads with 5-7 tweets using instructions for tone and flow.  
6. Parse AI JSON output with **Structured Output Parser** node.  
7. Append generated tweets/threads to “Other handles” Google Sheet with status “Waiting”.  
8. Mark original LinkedIn post as “All Done”.  
9. Notify user via Telegram.

---

**5. Automated Publishing**  
- **LinkedIn Posting:**  
1. Create a **Schedule Trigger** node (“6 PM”).  
2. Fetch LinkedIn posts with status “Waiting” via Google Sheets.  
3. Download post image from Google Drive.  
4. Post content with LinkedIn API node including text and image.  
5. Update post status to “Posted” with timestamp.  
6. Notify user.

- **X Posting:**  
1. Create a **Schedule Trigger** node (“6:30 PM”).  
2. Fetch X posts with status “Waiting” or “Ready”.  
3. Conditional check for image validity.  
4. If valid, upload image via Twitter OAuth 1.0a media endpoint.  
5. Post tweet with media or text-only if no valid image.  
6. Update status with post date.  
7. Notify user.

---

**6. Telegram Integration and On-Demand Commands**  
1. Add **Telegram Trigger** node configured for message commands and user ID filter.  
2. Add **Switch** node to parse commands: `/sms.scrape`, `/sms.linkedin:`, `/sms.x:`, else error.  
3. For each command, execute corresponding sub-workflow:  
   - Viral post scraping workflow  
   - LinkedIn post generation workflow  
   - X post generation workflow  
4. Add Telegram notification node to inform user of workflow start.  
5. Add error handling Telegram node for unauthorized users and invalid commands.

---

**7. Research and Verification (Optional)**  
1. Add sub-workflow invoking external scraping agents like browseract or Perplexity API to fetch factual info on demand.  
2. Integrate research results into AI content generation nodes for post enrichment.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                                                                                   |
|---------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| **System Description:** The workflow automates an end-to-end viral content engine for LinkedIn and X.    | Workflow title and purpose provided.                                                                             |
| **Setup Tips:** Update all “Configure Me” sticky notes with your API keys, sheet IDs, and chat IDs.      | Sticky notes in workflow guide these updates.                                                                    |
| **Scheduling:** Default scraping runs Saturdays 11 PM; LinkedIn posting Sundays 1 AM; X posting 7 AM.     | Adjust triggers as per your posting schedule.                                                                     |
| **Human-in-the-Loop:** Review content in Google Sheets and set status to “Ready” to enable auto-posting.  | Ensures quality control before publishing.                                                                        |
| **Telegram Bot:** Telegram integration allows manual control and notifications. Set your chat ID carefully.| Bot credentials and user ID must be secured.                                                                      |
| **AI Models:** Uses LangChain nodes with Google Gemini and OpenRouter models. Alternative OpenAI nodes exist.| Models can be switched with appropriate credential updates.                                                      |
| **Image Generation:** Uses Google Gemini for image generation with minimalistic style reference image.    | Optional OpenAI image generation nodes are present but disabled.                                                 |
| **Credentials:** Google Sheets, Drive, Telegram, Apify, OpenAI, Google Gemini, Twitter OAuth must be set. | Credential setup is critical.                                                                                      |
| **Research Tool:** Browseract agent used for external data research; Perplexity node included but disabled.| Helps ensure factual and credible content generation.                                                             |
| **Documentation:** Sticky notes provide detailed instructions, links, and reminders throughout workflow. | For example, Google Sheets template links and API docs.                                                          |
| **Workflow Modularity:** The workflow consists of multiple sub-workflows for scraping, generation, repurposing.| Modular design allows for independent updates and testing.                                                       |
| **Branding and Style:** AI prompts emphasize user’s tone, brand voice, and content buckets for consistency.| Custom prompts must be reviewed and adjusted to fit your voice and niche.                                        |
| **Error Handling:** Telegram messages notify about errors and unauthorized access attempts.              | Prevents misuse and ensures operator awareness.                                                                  |

---

### 3. Summary Table

| Node Name            | Type                         | Functional Role                              | Input Node(s)             | Output Node(s)            | Sticky Note                                              |
|----------------------|------------------------------|---------------------------------------------|---------------------------|---------------------------|----------------------------------------------------------|
| Manual Trigger        | Manual Trigger               | Start competitor scraping                    | None                      | Find Competitors           |                                                          |
| Edit Fields          | Set                         | Format competitor data                        | Find Competitors           | Add Competitors            |                                                          |
| Find Competitors      | HTTP Request                | Scrape LinkedIn profiles                      | Manual Trigger             | >100 likes                |                                                          |
| Add Competitors       | Google Sheets               | Append/update competitor info                 | Edit Fields                |                           |                                                          |
| >100 likes            | Filter                      | Filter posts by likes > 100                   | Find Competitors           | Add Competitors            |                                                          |
| Get Competitors       | Google Sheets               | Read competitor list                          | None                      | Scrape Viral Posts         |                                                          |
| Aggregate             | Aggregate                   | Collect LinkedIn URLs                         | Get Competitors            | Scrape Viral Posts         |                                                          |
| Saturday 11 PM        | Schedule Trigger            | Weekly scraping trigger                       | None                      | Get Competitors            |                                                          |
| Scrape Viral Posts    | HTTP Request                | Scrape viral posts                            | Aggregate                  | >100 likes                |                                                          |
| Remove Duplicates     | Remove Duplicates           | Remove duplicate posts                        | >100 likes                 | Text Classifier            |                                                          |
| Text Classifier       | LangChain Text Classifier   | Classify posts (EvergreenAI / Personal)      | Remove Duplicates           | Append Ideas               |                                                          |
| Append Ideas          | Google Sheets               | Append evergreen posts                        | Text Classifier             | Notify                    |                                                          |
| Notify                | Telegram                    | Notify user of scraping completion           | Append Ideas                |                           |                                                          |
| Sunday 1 AM           | Schedule Trigger            | Trigger LinkedIn post generation              | None                      | Get Viral Ideas            |                                                          |
| Get Viral Ideas       | Google Sheets               | Get posts marked “Ready”                      | None                      | Loop Over Items            |                                                          |
| Loop Over Items       | Split Batch                 | Process posts individually                    | Get Viral Ideas             | Post Repurpose             |                                                          |
| Post Repurpose        | LangChain Agent             | Rewrite posts and generate image description | Loop Over Items             | Gemini Flash               |                                                          |
| Research              | Sub-workflow                | Optional research for content enrichment      | Post Repurpose              | Post Repurpose             |                                                          |
| Gemini Flash          | LangChain Google Gemini     | Generate post image                           | Post Repurpose              | HTTP Request (Image Gen)   |                                                          |
| HTTP Request (Image)  | HTTP Request                | Call Gemini image API                         | Gemini Flash                | Convert to File            |                                                          |
| Minimalistic          | Google Drive Download       | Download style reference image                | None                      | HTTP Request (Image Gen)   |                                                          |
| Convert to File       | Convert to File             | Convert base64 image data to binary           | HTTP Request (Image Gen)   | Save Image                 |                                                          |
| Save Image            | Google Drive Upload         | Upload generated image                        | Convert to File             | Save Post                  |                                                          |
| Save Post             | Google Sheets               | Save generated post and image info            | Post Repurpose              | Mark Idea as Done          |                                                          |
| Mark Idea as Done     | Google Sheets               | Mark viral idea as processed                   | Save Post                  |                           |                                                          |
| Notify (Telegram)     | Telegram                    | Notify user of post generation status          | Save Post                  |                           |                                                          |
| Sunday 7 AM           | Schedule Trigger            | Trigger X (Twitter) repurposing                | None                      | Get Content                |                                                          |
| Get Content           | Google Sheets               | Get LinkedIn posts for repurposing             | None                      | Loop Over Items            |                                                          |
| Loop Over Items       | Split Batch                 | Process posts individually                    | Get Content                | Content Repurpose          |                                                          |
| Content Repurpose     | LangChain LLM Chain          | Generate X tweet (single)                      | Loop Over Items             | Structured Output Parser   |                                                          |
| Content Repurpose 1   | LangChain LLM Chain          | Generate X tweet thread                        | Loop Over Items             | Structured Output Parser   |                                                          |
| Structured Output Parser | LangChain Parser           | Parse AI JSON output                           | Content Repurpose           | Append                    |                                                          |
| Append                | Google Sheets               | Append X posts and status                      | Structured Output Parser    | Mark Idea as Done          |                                                          |
| Mark Idea as Done     | Google Sheets               | Mark LinkedIn post as done                      | Append                     | Notify                    |                                                          |
| Notify (Telegram)     | Telegram                    | Notify user about repurposed content            | Mark Idea as Done           |                           |                                                          |
| 6 PM                  | Schedule Trigger            | Trigger LinkedIn auto-posting                   | None                      | Get Content (LinkedIn)     |                                                          |
| Get Content (LinkedIn)| Google Sheets               | Get LinkedIn posts with status “Waiting”        | None                      | Get File                   |                                                          |
| Get File              | Google Drive Download       | Download LinkedIn post image                     | Get Content                | Image Valid?               |                                                          |
| Image Valid?          | If                         | Check if image is valid                          | Get File                   | Upload Media / Create Post |                                                          |
| Upload Media (X)      | HTTP Request               | Upload media to X platform                        | Image Valid? (true)         | Create Tweet               |                                                          |
| Create Post (LinkedIn)| LinkedIn API              | Publish post with image                          | Image Valid? / Get File     | Update Status              |                                                          |
| Create Tweet (X)      | Twitter API               | Post tweet with or without media                  | Upload Media / Image Valid? | Update Status              |                                                          |
| Update Status         | Google Sheets               | Update post status and date                       | Create Post / Create Tweet  | Notify                    |                                                          |
| Notify (Telegram)     | Telegram                    | Notify user on successful posting                  | Update Status              |                           |                                                          |
| Telegram Trigger      | Telegram                    | Listen to Telegram commands                       | None                      | Switch                     |                                                          |
| Switch                | Switch                     | Route commands                                    | Telegram Trigger           | Execute Sub-workflows      |                                                          |
| Execute Sub-Workflows | Execute Workflow           | Trigger sub-workflows based on command            | Switch                    | Notify                    |                                                          |
| Notify Start          | Telegram                    | Notify user of workflow start                       | Execute Sub-Workflows      |                           |                                                          |
| Error Handling        | Telegram                    | Notify unauthorized users or invalid commands      | Switch                    |                           |                                                          |
| Research (Sub-workflow) | Sub-workflow              | External research for post enrichment             | Post Repurpose / Post Generator | Post Repurpose / Post Generator |                                                          |

---

### 5. General Notes & Resources

| Note Content                                                                                                        | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow designed with modular sub-workflows for scraping, generation, repurposing, and publishing.                  | Allows flexibility in updating parts independently.                                                |
| Google Sheets templates used to store competitors, viral ideas, generated content, and posting status.               | Template link available in sticky notes.                                                          |
| Telegram bot integration supports on-demand workflow triggering with user access control via chat ID.               | Secure your Telegram credentials and chat ID.                                                     |
| AI prompts emphasize user tone, content buckets (EvergreenAI, Personal/Other), and minimalistic image generation.    | Prompts customizable in LangChain nodes; review before use.                                      |
| Gemini image generation used with a minimalistic style reference image stored in Google Drive.                       | Optional OpenAI image generation nodes are present but disabled.                                  |
| External research via browseract agent or optional Perplexity API enhances factual accuracy and content depth.       | Research nodes configured as sub-workflows; enable as needed.                                    |
| Human-in-the-loop: content is reviewed manually in Google Sheets before setting “Ready” status to allow auto-posting.| Ensures quality and brand consistency.                                                            |
| Posting schedules: scraping on Saturdays 11 PM; LinkedIn posting Sunday 1 AM; X posting Sunday 6-6:30 PM (user time).| Adjust schedules to match your timezone and frequency.                                           |
| OAuth credentials required for LinkedIn, X (Twitter OAuth 1.0a for media upload and OAuth 2 for posting), and APIs.   | Credential setup is critical for successful execution.                                            |
| Workflow includes error handling for API failures, unauthorized Telegram access, and missing media scenarios.       | Use Telegram notifications to monitor workflow health.                                           |
| Sticky notes throughout provide configuration guidance, links to documentation, and operational reminders.         | Review all sticky notes during setup.                                                            |

---

### 4. Reproducing the Workflow from Scratch

**1. Setup Credentials:**  
- Google Sheets OAuth with access to your sheets.  
- Google Drive OAuth for storing images and reference files.  
- Telegram Bot API token and your user chat ID.  
- Apify API key for LinkedIn scraping.  
- OpenAI or OpenRouter API key (optional).  
- Google Gemini API credentials for image and text generation.  
- X (Twitter) OAuth credentials (OAuth 1.0a for media upload, OAuth 2 for posting text).  

---

**2. Build Initialization Block:**  
- Add **Manual Trigger** (disabled by default).  
- Add **HTTP Request** node “Find Competitors” with POST to Apify LinkedIn scraper; set JSON payload with your niche keywords. Use Apify API key.  
- Add **Filter** node “>100 likes” to filter posts with more than 100 likes.  
- Add **Set** node “Edit Fields” to extract author name, LinkedIn URL, and headline.  
- Add **Google Sheets** node “Add Competitors” to append/update competitor profiles into Competitors sheet (match on LinkedIn URL).  
- Add **Google Sheets** node “Get Competitors” to read competitor LinkedIn URLs.  
- Add **Aggregate** node to collect LinkedIn URLs into a list.  

---

**3. Build Viral Content Discovery Block:**  
- Add **Schedule Trigger** “Saturday 11 PM” weekly.  
- Connect to **Get Competitors** to fetch profiles.  
- Add **HTTP Request** “Scrape Viral Posts” calling Apify API to scrape posts from competitors’ LinkedIn URLs.  
- Add **Filter** “>100 likes” to filter viral posts with over 100 likes.  
- Add **Remove Duplicates** node to prevent duplicates.  
- Add **LangChain Text Classifier** node; configure prompt to classify posts as “EvergreenAI” or “Personal/Other”.  
- Add **Google Sheets** node “Append Ideas” to append evergreen posts to Viral Ideas sheet.  
- Add **Telegram** node to notify you with a message and sheet link.  

---

**4. Build Content Generation Block:**  
- Add **Schedule Trigger** “Sunday 1 AM”.  
- Add **Google Sheets** node “Get Viral Ideas” to fetch posts marked “Ready” for generation.  
- Add **Split Batch** “Loop Over Items” to process posts one by one.  
- Add **LangChain Agent** “Post Repurpose” with custom prompt instructing rewriting into your tone, content buckets, image description generation.  
- Configure optional **Research** sub-workflow for on-demand fact-checking (browseract agent).  
- Add **LangChain Gemini Flash** node to generate images using prompt and style reference.  
- Add **Google Drive Download** “Minimalistic” node to get style reference image.  
- Add **HTTP Request** node calling Gemini image API with prompt and style image.  
- Add **Convert to File** node to convert base64 image data to binary.  
- Add **Google Drive Upload** “Save Image” node to upload generated images.  
- Add **Google Sheets** node “Save Post” to append generated content and image URLs to Content Gen sheet with status “Waiting”.  
- Add **Google Sheets** node “Mark Idea as Done” to update Viral Ideas row to “All Done”.  
- Add **Telegram** node to notify completion.  

---

**5. Build Repurposing for X Block:**  
- Add **Schedule Trigger** “Sunday 7 AM”.  
- Add **Google Sheets** node “Get Content” to fetch LinkedIn posts with status “Posted”.  
- Add **Split Batch** “Loop Over Items”.  
- Add **LangChain LLM Chain** node “Content Repurpose” for single tweets with prompt for 280 chars max.  
- Add **LangChain LLM Chain** node “Content Repurpose 1” for thread generation (5–7 tweets).  
- Add **Structured Output Parser** node to parse JSON response.  
- Add **Google Sheets** “Append” node to append tweets/threads to “Other handles” sheet with status “Waiting”.  
- Add **Google Sheets** “Mark Idea as Done” node to update LinkedIn post status.  
- Add **Telegram** notification node.  

---

**6. Build Automated Publishing Block:**  

*LinkedIn*  
- Add **Schedule Trigger** “6 PM”.  
- Add **Google Sheets** “Get Content” node to get posts with status “Waiting”.  
- Add **Google Drive Download** “Get File” node to download images.  
- Add **LinkedIn API** node “Create Post” with text and image.  
- Add **Google Sheets** “Update Status” node to mark as “Posted” with date.  
- Add **Telegram** notification node.  

*X (Twitter)*  
- Add **Schedule Trigger** “6:30 PM”.  
- Add **Google Sheets** “Get Content” node for X posts with status “Waiting” or “Ready”.  
- Add **If** node to check image validity.  
- Add **HTTP Request** node to upload media to Twitter via OAuth 1.0a media endpoint if image valid.  
- Add **Twitter API** node “Create Tweet” to post with media or text-only.  
- Add **Google Sheets** “Update Status” node to mark as “Posted” with date.  
- Add **Telegram** notification node.  

---

**7. Build Telegram Integration Block:**  
- Add **Telegram Trigger** node configured to listen to messages and filter by your chat ID.  
- Add **Switch** node to route commands:  
  - `/sms.scrape` → execute scraping workflow.  
  - `/sms.linkedin:` → execute LinkedIn post generation workflow.  
  - `/sms.x:` → execute X post generation workflow.  
  - Default → send error message.  
- Add **Execute Workflow** nodes to call respective sub-workflows asynchronously.  
- Add **Telegram** node to notify user of workflow start or errors.  

---

**8. Optional Research and Verification:**  
- Implement sub-workflow invoking browseract API for web scraping based on AI queries.  
- Integrate results into AI generation nodes.  
- Optionally enable Perplexity API nodes for advanced paid research.

---

### 6. General Notes & Resources

| Note Content                                                                                             | Context or Link                                                                                             |
|--------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------|
| Google Sheets template includes tabs for competitors, viral ideas, content gen, and other handles.      | Template link shared in sticky notes (user to replace with own).                                           |
| Sticky notes provide configuration instructions, including API keys, folder IDs, and chat IDs.         | Review all sticky notes during setup to avoid misconfiguration.                                            |
| AI prompt customization is critical to maintain brand voice and content quality.                        | Adjust LangChain prompts in “Post Repurpose”, “Content Repurpose”, and classifier nodes accordingly.       |
| Gemini image generation requires a style reference image stored in Google Drive.                        | Replace with your own minimalistic style image if desired.                                                 |
| Telegram bot commands enable manual control and notifications.                                         | Secure your Telegram credentials and restrict bot access via chat ID filtering.                            |
| Workflow scheduling can be modified to suit user’s timezone and posting preferences.                    | Default schedules are Saturday scraping, Sunday generation, and evening posting.                           |
| Human-in-the-loop content review is recommended before changing post statuses to “Ready” for publishing.| Provides quality control and prevents accidental posting of low-quality or off-brand content.              |
| Credentials for all external services (Google, Telegram, Apify, OpenAI, Gemini, Twitter) must be set.   | Use n8n’s credential manager to securely store keys and tokens.                                           |
| Workflow includes error handling with Telegram alerts for failures and unauthorized usage attempts.     | Monitor Telegram for operational issues during execution.                                                  |
| Modular design allows replacing AI providers or scraping services by adjusting corresponding nodes.    | For example, swap LangChain models or replace Apify scrapers if needed.                                    |
| Workflow documentation and video tutorials referenced in sticky notes and comments.                     | Users should consult provided resources for detailed setup help.                                         |

---

**Disclaimer:**  
This documentation is based exclusively on the provided n8n workflow JSON. It respects current usage policies and does not include any illegal or offensive content. All data handled is public or user-provided.