Scrape Blog Articles into AI-Generated LinkedIn Posts with GPT-4o & Human Review

https://n8nworkflows.xyz/workflows/scrape-blog-articles-into-ai-generated-linkedin-posts-with-gpt-4o---human-review-7310


# Scrape Blog Articles into AI-Generated LinkedIn Posts with GPT-4o & Human Review

### 1. Workflow Overview

This n8n workflow automatically scrapes newly published articles from the official n8n blog, generates engaging LinkedIn post drafts using GPT-4o mini, facilitates human review and editing via gotoHuman, and upon approval, publishes the posts to LinkedIn. It is designed for social media managers or content teams who want to streamline transforming blog content into polished LinkedIn posts with AI assistance and human oversight.

The workflow consists of the following logical blocks:

- **1.1 Scheduled Feed Retrieval:** Periodically fetches the n8n blog homepage HTML content and extracts individual article snippets.
- **1.2 Article Extraction & Filtering:** Splits the list of articles, extracts titles, links, tags, and publishing dates, and filters for articles published within the last 24 hours.
- **1.3 Article Content Processing:** Fetches a clean, LLM-friendly version of the article content, then summarizes it using GPT-4o mini.
- **1.4 LinkedIn Post Drafting:** Uses the summarized article to generate a LinkedIn post draft with GPT-4o mini, driven by a predefined prompt.
- **1.5 Human Review Loop:** Sends the drafted post to gotoHuman for human review and possible editing or retry requests.
- **1.6 Post Publishing:** Publishes the approved posts to LinkedIn.
- **1.7 Control Flow Management:** Handles looping over multiple new articles and retrying drafts based on human feedback.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Feed Retrieval

- **Overview:**  
This block triggers the workflow daily at 7 AM, fetches the HTML content of the n8n blog homepage, and extracts each blog post’s HTML snippet for further processing.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Fetch n8n blog feed  
  - Fetch feed item HTML  
  - Split Out

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow daily at 7:00 AM  
    - Configuration: Trigger hour set to 7  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Fetch n8n blog feed"  
    - Edge cases: Workflow will not run outside the scheduled time; no error expected here.

  - **Fetch n8n blog feed**  
    - Type: HTTP Request  
    - Role: Downloads the n8n blog homepage HTML  
    - Configuration: URL set to `https://blog.n8n.io/`, default HTTP GET  
    - Inputs: From Schedule Trigger  
    - Outputs: Connects to "Fetch feed item HTML"  
    - Edge cases: Network errors, HTTP errors, or site downtime could cause failure.

  - **Fetch feed item HTML**  
    - Type: HTML Extract  
    - Role: Parses the blog homepage HTML to extract article snippets  
    - Configuration: Extract CSS selector `article.item`, returns array of HTML snippets as "posts"  
    - Inputs: HTML from blog feed HTTP request  
    - Outputs: Connects to "Split Out"  
    - Edge cases: Changes in blog HTML structure may break extraction.

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits the array of posts into individual items, each representing one article snippet  
    - Configuration: Splits "posts" field into separate items  
    - Inputs: The array of article snippets  
    - Outputs: Connects to "Extract Title,Link+Tag"  
    - Edge cases: Empty post list leads to no further downstream activity.

---

#### 2.2 Article Extraction & Filtering

- **Overview:**  
Extracts title, link, and tags from each article snippet, constructs full article URLs, fetches full article HTML, extracts the publishing datetime, and filters to process only articles published within the last 24 hours.

- **Nodes Involved:**  
  - Extract Title,Link+Tag  
  - Article link (Set)  
  - Fetch Article HTML  
  - Extract publishing datetime  
  - Post from last 24hrs? (If)

- **Node Details:**

  - **Extract Title,Link+Tag**  
    - Type: HTML Extract  
    - Role: From each article snippet, extracts:  
      - link (href attribute of `.item-title a`)  
      - title (text of `.item-title a`)  
      - tags (array of `.item-tags a` texts)  
    - Inputs: Single article snippet from Split Out  
    - Outputs: Connects to "Article link"  
    - Edge cases: If selectors change or missing elements, values may be empty.

  - **Article link**  
    - Type: Set  
    - Role: Constructs the full article URL by prefixing extracted link with `https://blog.n8n.io`  
    - Assigns new field `articleUrl`  
    - Inputs: Output of "Extract Title,Link+Tag"  
    - Outputs: Connects to "Fetch Article HTML"  
    - Edge cases: If link is relative or malformed, URL construction may fail.

  - **Fetch Article HTML**  
    - Type: HTTP Request  
    - Role: Fetches full HTML of the individual article page  
    - Configuration: URL from `articleUrl` field  
    - Inputs: Output of "Article link"  
    - Outputs: Connects to "Extract publishing datetime"  
    - Edge cases: Network errors, 404 if article removed, or rate limits.

  - **Extract publishing datetime**  
    - Type: HTML Extract  
    - Role: Extracts article publish datetime from meta tag `meta[property="article:published_time"]` content attribute  
    - Outputs: Connects to "Post from last 24hrs?"  
    - Edge cases: Missing or malformed meta tag leads to no datetime extracted.

  - **Post from last 24hrs?**  
    - Type: If  
    - Role: Checks if article was published within the last 24 hours (currently set to 480 hours for testing)  
    - Condition: Compares difference between workflow run timestamp and published datetime  
    - True output: Connects to "Fetch Article (LLM-friendly)"  
    - False output: Skips downstream processing  
    - Edge cases: Timezone or datetime parsing issues may cause misfiltering.

---

#### 2.3 Article Content Processing

- **Overview:**  
Uses an external LLM-friendly scraper to fetch simplified article content, then summarizes it with GPT-4o mini.

- **Nodes Involved:**  
  - Fetch Article (LLM-friendly)  
  - Summarize Article  
  - Loop Over Items

- **Node Details:**

  - **Fetch Article (LLM-friendly)**  
    - Type: HTTP Request  
    - Role: Fetches a cleaned version of article content from `https://r.jina.ai/{articleUrl}`  
    - Inputs: Article URL from "Post from last 24hrs?"  
    - Outputs: Connects to "Summarize Article"  
    - Edge cases: Service unavailability, network errors, or invalid URLs.

  - **Summarize Article**  
    - Type: Langchain Chain LLM  
    - Role: Uses GPT-4o mini to summarize article content into key topics, news, trends, or best practices in markdown format  
    - Prompt: "You will be passed a scraped article from a company blog. Please summarize it to highlight key topics, news, trends, or best practices. Make sure to use markdown format in your response!"  
    - Inputs: Clean article text from "Fetch Article (LLM-friendly)"  
    - Outputs: Connects to "Loop Over Items"  
    - Edge cases: API errors, rate limits, or malformed input.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes multiple new articles one-by-one to ensure sequential human review  
    - Inputs: Summarized articles  
    - Outputs: Two outputs, main and second branch (which goes to "Set Initial Prompt")  
    - Edge cases: Empty input leads to no loops; batch size not configured explicitly (defaults apply).

---

#### 2.4 LinkedIn Post Drafting

- **Overview:**  
Sets the initial prompt for post drafting, generates a LinkedIn post draft using GPT-4o mini based on the summary, and prepares it for human review.

- **Nodes Involved:**  
  - Set Initial Prompt  
  - Draft LinkedIn Post  
  - OpenAI Chat Model1

- **Node Details:**

  - **Set Initial Prompt**  
    - Type: Set  
    - Role: Defines the prompt template for drafting LinkedIn posts:  
      "You will be passed a scraped article from a company blog. I'm an influencer for their product and will write a LinkedIn post for every news that is coming out. Please write an engaging post based on the article."  
    - Inputs: From "Loop Over Items" (secondary output)  
    - Outputs: Connects to "Draft LinkedIn Post"  
    - Edge cases: None significant; prompt can be customized.

  - **Draft LinkedIn Post**  
    - Type: Langchain Chain LLM  
    - Role: Uses GPT-4o mini to generate the LinkedIn post draft using the provided prompt and summary content  
    - Inputs: Text from fetched article (LLM-friendly), prompt from "Set Initial Prompt"  
    - Outputs: Connects to "Human approval"  
    - Edge cases: API errors, rate limits, or invalid prompt input.

  - **OpenAI Chat Model1**  
    - Type: LLM Chat OpenAI  
    - Role: Underlying GPT-4o mini model used by the Draft LinkedIn Post node  
    - Inputs/Outputs: Internal to Langchain nodes  
    - Edge cases: Requires valid OpenAI credentials; API quota limits apply.

---

#### 2.5 Human Review Loop

- **Overview:**  
Sends the AI-generated draft to gotoHuman for human review, then branches based on the review outcome: approved, rejected, or retry requested.

- **Nodes Involved:**  
  - Human approval (gotoHuman)  
  - Switch  
  - Retry  
  - Draft LinkedIn Post (looped via Retry)  
  - Loop Over Items (for rejected or post-processing)

- **Node Details:**

  - **Human approval**  
    - Type: gotoHuman review node  
    - Role: Sends draft post, summary, article link, and timestamp for human review  
    - Configuration: Uses a specific review template ID (`sMxevC9tSAgdfWsr6XIW`), fields mapped for link, summary, post draft, timestamp  
    - Credentials: Requires gotoHuman API key  
    - Inputs: Drafted post and context  
    - Outputs: Connects to "Switch" for decision branching  
    - Edge cases: Network errors, missing credentials, or template misconfiguration may cause failure.

  - **Switch**  
    - Type: Switch  
    - Role: Routes workflow based on human review response status:  
      - "approved" → "Create a post" node  
      - "rejected" → loops back to "Loop Over Items" (skips publishing)  
      - "Retry" → "Retry" node to redo drafting with updated prompt or edits  
    - Inputs: Review response from "Human approval"  
    - Outputs: Branches to respective nodes  
    - Edge cases: Unexpected or missing response values.

  - **Retry**  
    - Type: Set  
    - Role: Prepares the prompt and review ID for retrying the draft generation with updated instructions from human reviewer  
    - Inputs: From "Switch" Retry branch  
    - Outputs: Connects back to "Draft LinkedIn Post" for regeneration  
    - Edge cases: Missing or malformed retry instructions could cause loop failures.

---

#### 2.6 Post Publishing

- **Overview:**  
Publishes the human-approved LinkedIn post to the LinkedIn platform with relevant metadata.

- **Nodes Involved:**  
  - Create a post (LinkedIn)

- **Node Details:**

  - **Create a post**  
    - Type: LinkedIn  
    - Role: Publishes the approved LinkedIn post text with article title and URL as metadata  
    - Configuration:  
      - Text: LinkedIn post draft text from human approval result  
      - Person: (Empty string, presumably uses default or configured LinkedIn account credentials)  
      - Additional fields: Title from article, original URL  
      - Share media category: ARTICLE  
    - Inputs: From "Switch" Approved branch  
    - Outputs: Loops back to "Loop Over Items" to process next article  
    - Credentials: Requires LinkedIn OAuth2 credentials  
    - Edge cases: OAuth token expiration, API rate limits, LinkedIn API errors.

---

### 3. Summary Table

| Node Name               | Node Type                           | Functional Role                                         | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                   |
|-------------------------|-----------------------------------|---------------------------------------------------------|----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| Schedule Trigger         | Schedule Trigger                  | Starts workflow daily at 7AM                             | None                             | Fetch n8n blog feed               |                                                                                              |
| Fetch n8n blog feed      | HTTP Request                     | Fetches blog homepage HTML                               | Schedule Trigger                 | Fetch feed item HTML              | ## Fetch n8n blog feed: Reads the n8n blog page every day and extracts article items         |
| Fetch feed item HTML     | HTML Extract                     | Extracts article HTML snippets from blog homepage       | Fetch n8n blog feed              | Split Out                        |                                                                                              |
| Split Out                | Split Out                       | Splits article array into individual article items      | Fetch feed item HTML             | Extract Title,Link+Tag            | ## Fetch each article: Extracts title, link, tags from snippets, filters by publishing time  |
| Extract Title,Link+Tag   | HTML Extract                     | Extracts title, link, tags from article snippet         | Split Out                      | Article link                    |                                                                                              |
| Article link             | Set                              | Constructs full article URL                              | Extract Title,Link+Tag           | Fetch Article HTML               |                                                                                              |
| Fetch Article HTML       | HTTP Request                     | Fetches full article HTML                                | Article link                   | Extract publishing datetime      |                                                                                              |
| Extract publishing datetime | HTML Extract                  | Extracts article publish datetime                        | Fetch Article HTML             | Post from last 24hrs?            |                                                                                              |
| Post from last 24hrs?    | If                               | Filters articles published within last 24 hours         | Extract publishing datetime    | Fetch Article (LLM-friendly)     | ### ToDo: Change to `<=24`; currently set to 480 hours for testing                          |
| Fetch Article (LLM-friendly) | HTTP Request                | Fetches cleaned article content for LLM processing       | Post from last 24hrs? (true)   | Summarize Article               |                                                                                              |
| Summarize Article        | Langchain Chain LLM              | Summarizes article content using GPT-4o mini            | Fetch Article (LLM-friendly)   | Loop Over Items                 |                                                                                              |
| Loop Over Items          | Split In Batches                 | Processes multiple articles one-by-one                   | Summarize Article              | Set Initial Prompt (second output), or none | ## Loop over items: Process articles sequentially due to human review constraints           |
| Set Initial Prompt       | Set                              | Defines prompt template for LinkedIn post drafting      | Loop Over Items (secondary)     | Draft LinkedIn Post             | ## Create LinkedIn Post with a Human in the Loop: LLM drafts post for human review          |
| Draft LinkedIn Post      | Langchain Chain LLM              | Drafts LinkedIn post using GPT-4o mini                   | Set Initial Prompt             | Human approval                 |                                                                                              |
| Human approval           | gotoHuman node                   | Sends draft for human review and collects response      | Draft LinkedIn Post            | Switch                        |                                                                                              |
| Switch                  | Switch                          | Routes based on human review: approved, rejected, retry | Human approval                 | Create a post / Loop Over Items / Retry |                                                                                              |
| Retry                   | Set                              | Prepares retry prompt and review ID for re-drafting     | Switch (Retry branch)          | Draft LinkedIn Post             |                                                                                              |
| Create a post            | LinkedIn                        | Publishes approved post to LinkedIn                      | Switch (Approved branch)       | Loop Over Items                 | ## Publish approved post: After human approval, submit to LinkedIn                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Configure to trigger daily at 7:00 AM.

2. **Create HTTP Request Node for Blog Feed:**  
   - Connect from Schedule Trigger.  
   - Set URL to `https://blog.n8n.io/`.  
   - Method: GET (default).

3. **Create HTML Extract Node to Extract Article Snippets:**  
   - Connect from Blog Feed HTTP Request.  
   - Operation: extractHtmlContent  
   - Extraction: CSS selector `article.item`, return array of HTML snippets as `posts`.

4. **Create Split Out Node:**  
   - Connect from HTML Extract.  
   - Field to split out: `posts`.  
   - Destination field name: `post`.

5. **Create HTML Extract Node to Extract Title, Link, and Tags:**  
   - Connect from Split Out.  
   - Operation: extractHtmlContent from `post` field.  
   - Extract:  
     - `link` attribute from `.item-title a` (href)  
     - `title` text from `.item-title a`  
     - `tags` array of text from `.item-tags a`.

6. **Create Set Node to Construct Full Article URL:**  
   - Connect from previous node.  
   - Set new field: `articleUrl`, value: `https://blog.n8n.io{{ $json.link }}`.  
   - Include other fields.

7. **Create HTTP Request Node to Fetch Full Article HTML:**  
   - Connect from Article link Set node.  
   - URL: `={{ $json.articleUrl }}`.

8. **Create HTML Extract Node to Extract Publishing Datetime:**  
   - Connect from article HTTP request.  
   - Extract content attribute of `meta[property="article:published_time"]` as `published`.

9. **Create If Node to Filter Articles Published Within Last 24 Hours:**  
   - Connect from publishing datetime extraction.  
   - Condition:  
     - Calculate difference between current execution timestamp (`$('Schedule Trigger').item.json.timestamp`) and `published` date.  
     - Accept if difference ≤ 24 hours (currently set to 480 hours for testing, should be changed to 24).  
   - True branch: continue; False branch: stop.

10. **Create HTTP Request Node to Fetch LLM-Friendly Article Content:**  
    - Connect from If node true branch.  
    - URL: `https://r.jina.ai/{{ $json.articleUrl }}`.

11. **Create Langchain Chain LLM Node to Summarize Article:**  
    - Connect from LLM-friendly fetch.  
    - Model: GPT-4o mini.  
    - Prompt: "You will be passed a scraped article from a company blog. Please summarize it to highlight key topics, news, trends, or best practices. Make sure to use markdown format in your response!"  
    - Input text: the fetched article content.

12. **Create Split In Batches Node to Loop Over Articles:**  
    - Connect from Summarize Article.  
    - Configure default batch size (e.g., 1).  
    - This allows sequential processing for human review.

13. **Create Set Node for Initial Prompt:**  
    - Connect from Loop Over Items (second output).  
    - Set field `prompt` with value:  
      "You will be passed a scraped article from a company blog. I'm an influencer for their product and will write a LinkedIn post for every news that is coming out. Please write an engaging post based on the article."

14. **Create Langchain Chain LLM Node to Draft LinkedIn Post:**  
    - Connect from Set Initial Prompt.  
    - Model: GPT-4o mini.  
    - Input text: article content from LLM-friendly fetch (`$('Fetch Article (LLM-friendly)').item.json.data`).  
    - Input prompt: from `prompt` field set above.

15. **Create gotoHuman Node for Human Approval:**  
    - Connect from Draft LinkedIn Post.  
    - Configure:  
      - API credentials for gotoHuman.  
      - Review template ID `sMxevC9tSAgdfWsr6XIW` (import this template in gotoHuman beforehand).  
      - Map fields for article link, summary, post draft, timestamp.  
      - Allow update for review ID on retry.

16. **Create Switch Node to Branch on Human Review Result:**  
    - Connect from Human approval.  
    - Conditions:  
      - `response == "approved"` → Connect to "Create a post" node  
      - `response == "rejected"` → Connect to Loop Over Items (to skip/post-processing)  
      - `type == "chat"` (retry request) → Connect to Retry node.

17. **Create Set Node for Retry Handling:**  
    - Connect from Switch retry branch.  
    - Set fields:  
      - `prompt` to the updated prompt content from human review.  
      - `reviewToUpdate` with review ID for updating.  
    - Connect output to Draft LinkedIn Post node (loop retry).

18. **Create LinkedIn Node to Publish Approved Posts:**  
    - Connect from Switch approved branch.  
    - Configure LinkedIn OAuth2 credentials.  
    - Set post text from human-approved draft (`$json.responseValues.postDraft.value`).  
    - Set additional fields: title and original URL from article metadata.  
    - Set media category as ARTICLE.

19. **Connect "Create a post" node output back to Loop Over Items:**  
    - This allows processing the next article after publishing.

---

### 5. General Notes & Resources

| Note Content                                                                                                                   | Context or Link                                                                                                                       |
|-------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------------------------------|
| The 24h filter condition is currently set to 480 hours for testing; change to `<=24` to restrict to posts from the last 24h.  | See Sticky Note near "Post from last 24hrs?" node                                                                                    |
| Human review uses gotoHuman platform; requires setup of API key and importing review template `sMxevC9tSAgdfWsr6XIW`.           | https://docs.gotohuman.com/Integrations/n8n#multiple-items-input                                                                     |
| Looping over multiple articles is sequential to avoid simultaneous human reviews, per gotoHuman best practices.                | https://docs.gotohuman.com/Integrations/n8n#multiple-items-input                                                                     |
| OpenAI GPT-4o mini is used for summarization and drafting; ensure valid OpenAI API credentials configured in n8n.             |                                                                                                                                    |
| LinkedIn OAuth2 credentials are required for posting.                                                                           |                                                                                                                                    |
| The workflow uses a free Jina.ai scraping service (`https://r.jina.ai/`) to get LLM-friendly article content.                  | Service availability may vary; consider fallback or error handling.                                                                  |
| This workflow is branded "n8n News Assistant" and designed to automatically detect news from n8n blog and output LinkedIn posts. | See Sticky Note with setup instructions and branding images embedded in the workflow canvas.                                         |

---

**Disclaimer:**  
The text provided exclusively originates from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.