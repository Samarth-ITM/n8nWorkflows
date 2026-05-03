Promote YouTube Videos on Reddit with AI-Generated Comments and Email Digest

https://n8nworkflows.xyz/workflows/promote-youtube-videos-on-reddit-with-ai-generated-comments-and-email-digest-4433


# Promote YouTube Videos on Reddit with AI-Generated Comments and Email Digest

---

### 1. Workflow Overview

This n8n workflow automates the promotion of YouTube videos on Reddit by generating AI-powered comments and compiling an email digest of activities. It integrates YouTube data retrieval, AI-based content analysis and generation, Reddit interaction, Google Sheets data management, and email notifications.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception**  
  Captures new YouTube video information triggered by a form submission.

- **1.2 YouTube Video Analysis and Keyword Extraction**  
  Retrieves YouTube videos and analyzes keywords using AI agents.

- **1.3 Reddit Posts Retrieval and Filtering**  
  Fetches related Reddit posts, removes duplicates, and filters posts based on criteria.

- **1.4 AI Relevance Assessment and Comment Generation**  
  Uses AI to determine the relevance of Reddit posts and generate humanized comments.

- **1.5 Data Storage and Management**  
  Stores generated comments and appends data to Google Sheets.

- **1.6 Email Digest Generation and Sending**  
  Compiles and sends an HTML email digest summarizing the promotional activities.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
  This block initiates the workflow by receiving input via a form trigger node.

- **Nodes Involved:**  
  - Form  
  - YouTube

- **Node Details:**  
  - **Form**  
    - *Type:* Trigger  
    - *Role:* Entry point capturing user input or trigger event to start workflow  
    - *Configuration:* Uses a webhook with a defined webhook ID to listen for submissions  
    - *Input:* Incoming webhook payload (form data)  
    - *Output:* Triggers the YouTube node  
    - *Edge Cases:* Webhook failures, missing expected form fields  

  - **YouTube**  
    - *Type:* YouTube node  
    - *Role:* Fetches YouTube video data based on form input  
    - *Configuration:* Uses credentials set up for YouTube API access  
    - *Input:* Trigger from Form node, expects video or channel identifiers  
    - *Output:* Passes video data to AI Keyword Analyst node  
    - *Edge Cases:* API quota exceeded, invalid video IDs  

#### 2.2 YouTube Video Analysis and Keyword Extraction

- **Overview:**  
  This block analyzes YouTube video data to extract relevant keywords for Reddit promotion using AI agents.

- **Nodes Involved:**  
  - YT Keyword Analyst  
  - AI Brain  
  - Structured Output  
  - Split Out

- **Node Details:**  
  - **AI Brain**  
    - *Type:* Langchain LM Chat Open Router (AI Language Model)  
    - *Role:* Provides AI processing capability to analyze YouTube video metadata and generate keyword suggestions  
    - *Input:* YouTube video data  
    - *Output:* Passed to YT Keyword Analyst via AI language model interface  
    - *Edge Cases:* API rate limits, connection timeouts, malformed input data  

  - **YT Keyword Analyst**  
    - *Type:* Langchain Agent  
    - *Role:* Processes AI Brain output to identify keywords and relevant tags  
    - *Input:* AI Brain processed data via AI language model  
    - *Output:* Outputs structured keyword data to Split Out and Structured Output nodes  
    - *Edge Cases:* Parsing errors, incomplete AI responses  

  - **Structured Output**  
    - *Type:* Langchain Output Parser Structured  
    - *Role:* Parses AI-generated outputs into structured JSON format for downstream processing  
    - *Input:* AI outputs from YT Keyword Analyst  
    - *Output:* Feeds structured data back into YT Keyword Analyst  
    - *Edge Cases:* Parsing failures due to unexpected AI response formats  

  - **Split Out**  
    - *Type:* Split Out  
    - *Role:* Splits keyword data into individual items for further processing  
    - *Input:* Keywords array from YT Keyword Analyst  
    - *Output:* Sends each keyword item to Reddit node  
    - *Edge Cases:* Empty arrays, malformed inputs  

#### 2.3 Reddit Posts Retrieval and Filtering

- **Overview:**  
  This block retrieves Reddit posts related to the extracted keywords, removes duplicates, and filters posts according to criteria to find relevant discussion threads.

- **Nodes Involved:**  
  - Reddit  
  - Remove Duplicates  
  - Filter Posts by Criteria  
  - Keep Certain Fields

- **Node Details:**  
  - **Reddit**  
    - *Type:* Reddit node  
    - *Role:* Queries Reddit API for posts matching keywords  
    - *Input:* Keywords from Split Out node  
    - *Output:* Raw Reddit posts data forwarded to Remove Duplicates  
    - *Edge Cases:* API authentication errors, rate limiting, no results found  

  - **Remove Duplicates**  
    - *Type:* Remove Duplicates  
    - *Role:* Eliminates duplicate Reddit posts to avoid redundant processing  
    - *Input:* Reddit posts data  
    - *Output:* Unique posts sent to Filter Posts by Criteria  
    - *Edge Cases:* Improper deduplication if keys are not well-defined  

  - **Filter Posts by Criteria**  
    - *Type:* If (Conditional)  
    - *Role:* Applies business logic filters to choose posts relevant for promotion (e.g., post age, subreddit, engagement level)  
    - *Input:* Unique Reddit posts  
    - *Output:* Posts matching criteria forwarded to Keep Certain Fields  
    - *Edge Cases:* Incorrect filtering logic leading to empty result sets  

  - **Keep Certain Fields**  
    - *Type:* Set  
    - *Role:* Selects and retains only essential data fields from Reddit posts for further processing  
    - *Input:* Filtered Reddit posts  
    - *Output:* Sends cleaned posts to Loop Over Items node  
    - *Edge Cases:* Missing expected fields in input data  

#### 2.4 AI Relevance Assessment and Comment Generation

- **Overview:**  
  This block uses AI to assess if Reddit posts are relevant for promotion and generates humanized comments to post.

- **Nodes Involved:**  
  - Loop Over Items  
  - Get Reddit Posts & Proposed Responses  
  - If  
  - Brain  
  - Structured Output Parser  
  - Is this post relevant?  
  - relevant?  
  - Wait 5 sec  
  - Append Data  
  - Social Post Comment  
  - Structured_Output  
  - Brain1  
  - Store Humanized Comment

- **Node Details:**  
  - **Loop Over Items**  
    - *Type:* Split In Batches  
    - *Role:* Iterates over each filtered Reddit post for individual processing  
    - *Input:* Posts from Keep Certain Fields  
    - *Output:* Each post sent sequentially to AI relevance checks and comment generation  
    - *Edge Cases:* Batch size too large causing timeouts  

  - **Get Reddit Posts & Proposed Responses**  
    - *Type:* Google Sheets  
    - *Role:* Retrieves stored Reddit posts and previously proposed responses to avoid repeats  
    - *Input:* Batch loop iteration  
    - *Output:* Data sent to If node for conditional logic  
    - *Edge Cases:* Google Sheets API errors, empty sheets  

  - **If**  
    - *Type:* Conditional  
    - *Role:* Checks if the Reddit post has already been processed or meets additional conditions  
    - *Input:* Data from Google Sheets  
    - *Output:* Controls flow to AI relevance assessment or skips processing  
    - *Edge Cases:* Logic misconfiguration  

  - **Brain**  
    - *Type:* Langchain LM Chat Open Router  
    - *Role:* AI node assessing post relevance based on content and context  
    - *Input:* Reddit post details  
    - *Output:* Sent to Structured Output Parser  
    - *Edge Cases:* AI service failures, rate limits  

  - **Structured Output Parser**  
    - *Type:* Langchain Output Parser Structured  
    - *Role:* Parses AI relevance output into structured format  
    - *Input:* AI relevance raw output  
    - *Output:* Sent to Is this post relevant? node  
    - *Edge Cases:* Parsing errors  

  - **Is this post relevant?**  
    - *Type:* Langchain Agent  
    - *Role:* Final determination of post relevance to prepare comment generation  
    - *Input:* Parsed AI output  
    - *Output:* Sends boolean decision to relevant? node  
    - *Edge Cases:* AI response ambiguity  

  - **relevant?**  
    - *Type:* If  
    - *Role:* Routes workflow based on relevance decision  
    - *Input:* Boolean from Is this post relevant?  
    - *Output:* If relevant, proceeds to Wait 5 sec and Append Data; else loops back or skips  
    - *Edge Cases:* Incorrect routing  

  - **Wait 5 sec**  
    - *Type:* Wait  
    - *Role:* Implements delay to respect rate limits before posting and data insertion  
    - *Input:* Relevant posts only  
    - *Output:* Sends to Append Data  
    - *Edge Cases:* Workflow delays causing timeouts  

  - **Append Data**  
    - *Type:* Google Sheets  
    - *Role:* Appends new data entries such as post interactions or comment proposals into Google Sheets for tracking  
    - *Input:* Post data after waiting  
    - *Output:* Sends to Social Post Comment node  
    - *Edge Cases:* Google Sheets write errors  

  - **Social Post Comment**  
    - *Type:* Langchain Agent  
    - *Role:* Generates AI-crafted humanized comments tailored to each Reddit post  
    - *Input:* Appended data about posts  
    - *Output:* Sends generated comments to Store Humanized Comment  
    - *Edge Cases:* AI generation failures, output formatting issues  

  - **Structured_Output**  
    - *Type:* Langchain Output Parser Structured  
    - *Role:* Parses generated comment into structured data  
    - *Input:* Raw AI comment output  
    - *Output:* Feeds into Social Post Comment node (possibly a loop or confirmation)  

  - **Brain1**  
    - *Type:* Langchain LM Chat Open Router  
    - *Role:* Additional AI processing related to comment generation or refinement  
    - *Input:* Possibly feeds Social Post Comment or related nodes  
    - *Output:* AI-generated comment improvements or checks  
    - *Edge Cases:* AI service availability  

  - **Store Humanized Comment**  
    - *Type:* Google Sheets  
    - *Role:* Stores final AI-generated comments for record-keeping and auditability  
    - *Input:* Generated comments from Social Post Comment  
    - *Output:* Loops back to Loop Over Items for next item processing  
    - *Edge Cases:* Data integrity, concurrency issues  

#### 2.5 Data Storage and Management

- **Overview:**  
  Manages data persistence by recording processed posts and generated comments in Google Sheets for tracking and later use.

- **Nodes Involved:**  
  - Append Data  
  - Store Humanized Comment  
  - Get Reddit Posts & Proposed Responses

- **Node Details:**  
  - Covered in previous blocks; these nodes handle insertion and retrieval of data in Google Sheets. Proper credentials and API limits must be managed.  

#### 2.6 Email Digest Generation and Sending

- **Overview:**  
  Generates an HTML email summarizing the promotion efforts and sends it to a configured email address.

- **Nodes Involved:**  
  - Generate Email HTML  
  - Send to your email  
  - If

- **Node Details:**  
  - **If**  
    - *Type:* Conditional  
    - *Role:* Determines if an email should be generated and sent based on prior outcomes  
    - *Input:* Results from comment generation or other criteria  
    - *Output:* Triggers Generate Email HTML node if condition is met  
    - *Edge Cases:* Conditional logic errors  

  - **Generate Email HTML**  
    - *Type:* Code  
    - *Role:* Custom JavaScript code to construct an HTML email body summarizing posts and comments  
    - *Input:* Aggregated data from workflow execution  
    - *Output:* Feeds HTML content to Send to your email  
    - *Edge Cases:* Code errors, missing data fields  

  - **Send to your email**  
    - *Type:* Gmail node  
    - *Role:* Sends the generated email digest via Gmail SMTP using OAuth2 credentials  
    - *Input:* HTML email content  
    - *Output:* Final node in email sending branch  
    - *Edge Cases:* Authentication errors, email sending failures  

---

### 3. Summary Table

| Node Name                       | Node Type                               | Functional Role                                | Input Node(s)                 | Output Node(s)                    | Sticky Note                    |
|--------------------------------|---------------------------------------|------------------------------------------------|------------------------------|---------------------------------|-------------------------------|
| Form                           | Form Trigger                          | Workflow entry point capturing input          | -                            | YouTube                         |                               |
| YouTube                        | YouTube                              | Fetches YouTube video data                      | Form                         | AI Brain                       |                               |
| AI Brain                       | Langchain LM Chat Open Router        | AI analysis for keyword extraction             | YouTube                      | YT Keyword Analyst             |                               |
| YT Keyword Analyst             | Langchain Agent                      | Extracts keywords and tags                       | AI Brain                     | Split Out                     |                               |
| Structured Output              | Langchain Output Parser Structured   | Parses AI output into structured JSON           | YT Keyword Analyst           | YT Keyword Analyst             |                               |
| Split Out                     | Split Out                           | Splits keywords into items                        | YT Keyword Analyst           | Reddit                        |                               |
| Reddit                        | Reddit                              | Retrieves Reddit posts related to keywords       | Split Out                    | Remove Duplicates             |                               |
| Remove Duplicates             | Remove Duplicates                    | Removes duplicate Reddit posts                    | Reddit                       | Filter Posts by Criteria      |                               |
| Filter Posts by Criteria      | If (Conditional)                    | Filters posts based on business logic            | Remove Duplicates            | Keep Certain Fields           |                               |
| Keep Certain Fields           | Set                                | Selects essential fields from posts               | Filter Posts by Criteria     | Loop Over Items              |                               |
| Loop Over Items               | Split In Batches                   | Iterates over posts for individual processing     | Keep Certain Fields          | Get Reddit Posts & Proposed Responses, Is this post relevant? |                               |
| Get Reddit Posts & Proposed Responses | Google Sheets                     | Retrieves stored posts and proposed responses      | Loop Over Items              | If                           |                               |
| If                           | If (Conditional)                   | Checks if post already processed                   | Get Reddit Posts & Proposed Responses | Brain                   |                               |
| Brain                        | Langchain LM Chat Open Router       | AI assessment of post relevance                    | If                          | Structured Output Parser     |                               |
| Structured Output Parser     | Langchain Output Parser Structured  | Parses AI relevance output                          | Brain                       | Is this post relevant?       |                               |
| Is this post relevant?       | Langchain Agent                   | Determines if post is relevant                      | Structured Output Parser     | relevant?                    |                               |
| relevant?                    | If (Conditional)                   | Routes based on relevance                            | Is this post relevant?       | Wait 5 sec, Loop Over Items  |                               |
| Wait 5 sec                   | Wait                              | Delay to respect API rate limits                     | relevant?                   | Append Data                  |                               |
| Append Data                  | Google Sheets                     | Appends new post/comment data                         | Wait 5 sec                  | Social Post Comment          |                               |
| Social Post Comment          | Langchain Agent                  | Generates AI humanized comments                       | Append Data                 | Store Humanized Comment      |                               |
| Structured_Output            | Langchain Output Parser Structured  | Parses AI-generated comments                          | Social Post Comment         | Brain1                      |                               |
| Brain1                       | Langchain LM Chat Open Router       | Further AI processing/refinement of comments         | Structured_Output           | Social Post Comment          |                               |
| Store Humanized Comment      | Google Sheets                     | Stores final comments for records                      | Social Post Comment         | Loop Over Items              |                               |
| Generate Email HTML          | Code                              | Generates HTML email digest                            | If                          | Send to your email           |                               |
| Send to your email           | Gmail                             | Sends email digest via Gmail                           | Generate Email HTML          | -                           |                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node:**  
   - Type: Form Trigger  
   - Configure webhook ID for receiving user input to start workflow.

2. **Add YouTube Node:**  
   - Type: YouTube  
   - Connect from Form Trigger output.  
   - Configure with YouTube API credentials.  
   - Set to fetch video data based on form input (e.g., video ID).

3. **Add AI Brain Node:**  
   - Type: Langchain LM Chat Open Router  
   - Connect from YouTube output.  
   - Configure with OpenAI or similar AI credentials.  
   - Purpose: Analyze video data for keyword extraction.

4. **Add YT Keyword Analyst Node:**  
   - Type: Langchain Agent  
   - Connect AI Brain's language model output here.  
   - Configure prompt or agent to extract keywords related to video content.

5. **Add Structured Output Parser Node:**  
   - Type: Langchain Output Parser Structured  
   - Connect from YT Keyword Analyst AI output.  
   - Purpose: Parse AI response into structured JSON.

6. **Add Split Out Node:**  
   - Type: Split Out  
   - Connect from YT Keyword Analyst main output.  
   - Purpose: Split keyword list into individual items.

7. **Add Reddit Node:**  
   - Type: Reddit  
   - Connect from Split Out output.  
   - Configure Reddit API credentials.  
   - Query Reddit posts matching keywords.

8. **Add Remove Duplicates Node:**  
   - Type: Remove Duplicates  
   - Connect from Reddit output.  
   - Configure keys to identify duplicates (e.g., post ID).

9. **Add Filter Posts by Criteria Node (If):**  
   - Type: If (Conditional)  
   - Connect from Remove Duplicates output.  
   - Configure rules to filter posts (e.g., subreddit, post age, engagement).

10. **Add Keep Certain Fields Node (Set):**  
    - Type: Set  
    - Connect from Filter Posts by Criteria true branch.  
    - Configure to retain key fields required for next steps.

11. **Add Loop Over Items Node (Split In Batches):**  
    - Type: Split In Batches  
    - Connect from Keep Certain Fields output.  
    - Configure batch size as needed (e.g., 1 for sequential processing).

12. **Add Get Reddit Posts & Proposed Responses Node:**  
    - Type: Google Sheets  
    - Connect from Loop Over Items output.  
    - Configure to read stored data to check for previously processed posts.

13. **Add If Node:**  
    - Type: If (Conditional)  
    - Connect from Get Reddit Posts & Proposed Responses output.  
    - Configure to skip already processed posts.

14. **Add Brain Node (AI Relevance Assessment):**  
    - Type: Langchain LM Chat Open Router  
    - Connect from If true branch.  
    - Configure prompt to evaluate post relevance.

15. **Add Structured Output Parser Node:**  
    - Type: Langchain Output Parser Structured  
    - Connect from Brain AI output.

16. **Add Is this post relevant? Node:**  
    - Type: Langchain Agent  
    - Connect from Structured Output Parser.  
    - Configure to finalize relevance determination.

17. **Add relevant? Node (If):**  
    - Type: If (Conditional)  
    - Connect from Is this post relevant? output.  
    - True branch to Wait 5 sec node; false branch loops or ends.

18. **Add Wait 5 sec Node:**  
    - Type: Wait  
    - Connect from relevant? true branch.  
    - Set to 5 seconds to avoid rate limits.

19. **Add Append Data Node:**  
    - Type: Google Sheets  
    - Connect from Wait 5 sec node.  
    - Configure to append post/comment data for tracking.

20. **Add Social Post Comment Node:**  
    - Type: Langchain Agent  
    - Connect from Append Data output.  
    - Configure to generate AI humanized Reddit comments.

21. **Add Structured_Output Parser Node:**  
    - Type: Langchain Output Parser Structured  
    - Connect from Social Post Comment AI output.

22. **Add Brain1 Node:**  
    - Type: Langchain LM Chat Open Router  
    - Connect from Structured_Output output.  
    - Optionally refine comments.

23. **Connect Brain1 output back to Social Post Comment for potential retries or refinements.**

24. **Add Store Humanized Comment Node:**  
    - Type: Google Sheets  
    - Connect from Social Post Comment output.  
    - Configure to store final comments.

25. **Connect Store Humanized Comment output back to Loop Over Items for next item.**

26. **Add Generate Email HTML Node:**  
    - Type: Code  
    - Connect from If node that determines if email should be sent (conditional).  
    - Write custom JavaScript to build the email content summarizing posts and comments.

27. **Add Send to your email Node:**  
    - Type: Gmail  
    - Connect from Generate Email HTML output.  
    - Configure Gmail OAuth2 credentials.  
    - Set recipient and email parameters.

---

### 5. General Notes & Resources

| Note Content                                                                                     | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Workflow automates YouTube video promotion on Reddit with AI-generated comments and email digest | Project description                                                                              |
| Requires valid API credentials for YouTube, Reddit, Google Sheets, OpenAI (or compatible AI), Gmail | Credential setup steps must be completed prior to execution                                      |
| Rate limits and API quotas should be monitored to avoid workflow failures                        | Pay special attention to Wait node usage and batch sizes                                        |
| Langchain nodes require compatible AI services and careful prompt engineering                   | See Langchain documentation for prompt and output format guidance                               |
| Sticky notes present in workflow are empty, indicating no additional comments                    | No extra commentary from author                                                                  |

---

**Disclaimer:**  
The text provided is extracted exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects applicable content policies and contains no illegal, offensive, or protected elements. All handled data is legal and publicly accessible.

---