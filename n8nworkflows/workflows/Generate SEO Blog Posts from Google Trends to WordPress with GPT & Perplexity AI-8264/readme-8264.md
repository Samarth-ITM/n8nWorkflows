Generate SEO Blog Posts from Google Trends to WordPress with GPT & Perplexity AI

https://n8nworkflows.xyz/workflows/generate-seo-blog-posts-from-google-trends-to-wordpress-with-gpt---perplexity-ai-8264


# Generate SEO Blog Posts from Google Trends to WordPress with GPT & Perplexity AI

# Comprehensive Reference Document for n8n Workflow  
**Workflow Title:** Generate SEO Blog Posts from Google Trends to WordPress with GPT & Perplexity AI

---

## 1. Workflow Overview

This workflow automates the entire process of generating SEO-optimized blog posts by leveraging trending data, AI content generation, and WordPress publishing. It is designed for content marketers, SEO specialists, and digital agencies aiming for scalable, hands-free content creation targeting trending topics.

### Logical Blocks and Their Roles:

- **1.1 Trigger and Trend Detection:**  
  Scheduled trigger initiates the workflow periodically, fetching trending search queries from Google Trends via SerpAPI.

- **1.2 Trend Filtering and Topic Selection:**  
  Processes and filters trending queries by volume, then uses an AI agent to select the best SEO-relevant topic.

- **1.3 Content Research:**  
  Uses the Perplexity API to gather reliable source information on the selected topic.

- **1.4 Blog Drafting:**  
  AI generates a full blog post draft based on the researched content and trending keywords.

- **1.5 Internal Linking Preparation:**  
  Retrieves previous blog posts from Google Sheets and prepares them for internal link insertion.

- **1.6 Internal Link Insertion:**  
  AI analyzes previous posts and inserts relevant internal links into the new blog draft.

- **1.7 SEO Metadata and Formatting:**  
  Generates semantic HTML, SEO-friendly slug, blog title, and meta description with AI.

- **1.8 Cover Image Fetching:**  
  Retrieves a relevant cover image from Google Images based on the blog topic.

- **1.9 WordPress Publishing and Logging:**  
  Publishes the blog post as a draft on WordPress with metadata and cover image, then logs publication data to Google Sheets.

---

## 2. Block-by-Block Analysis

### 2.1 Trigger and Trend Detection

**Overview:**  
Initiates the workflow on a schedule and fetches recent Google Trends data for a specific keyword.

**Nodes Involved:**  
- Schedule Trigger  
- Get Google Trends  
- Extract Top 2 Trends  
- Filter High Volume Keywords

**Node Details:**

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Configuration:* Runs at a fixed interval (default unspecified, typically daily).  
  - *Inputs:* None (start node).  
  - *Outputs:* Triggers the next node.  
  - *Failure Modes:* Misconfiguration or scheduling errors.

- **Get Google Trends**  
  - *Type:* HTTP Request  
  - *Role:* Fetches Google Trends data via SerpAPI for the query "ai agent" limited to US region and last 3 days.  
  - *Configuration:* Uses HTTP Query Auth credentials for SerpAPI. Sends parameters for query (q), geography (geo), language (hl), date range (last 3 days), and data type "RELATED_QUERIES".  
  - *Input:* Trigger from Schedule Trigger.  
  - *Output:* JSON containing trends and related queries.  
  - *Edge Cases:* API rate limits, authentication failure, network timeout.

- **Extract Top 2 Trends**  
  - *Type:* Set Node  
  - *Role:* Extracts the top 2 rising related queries with their query text and score.  
  - *Configuration:* Uses expressions to access the first two entries from `related_queries.rising`.  
  - *Input:* From "Get Google Trends".  
  - *Output:* JSON object with two top trending keywords and their scores.  
  - *Edge Cases:* Empty or missing `related_queries.rising` array, null values.

- **Filter High Volume Keywords**  
  - *Type:* Code (JavaScript)  
  - *Role:* Filters "top" related queries to those with volume (extracted_value) greater than 30, outputs their queries as a comma-separated string.  
  - *Input:* From "Get Google Trends".  
  - *Output:* JSON with filtered keyword string.  
  - *Edge Cases:* Empty top queries, missing `extracted_value`, runtime exceptions in code.

---

### 2.2 Trend Filtering and Topic Selection

**Overview:**  
Selects the single best SEO topic from the top two trends using an AI model considering relevance and trendiness.

**Nodes Involved:**  
- Select Best SEO Topic

**Node Details:**

- **Select Best SEO Topic**  
  - *Type:* OpenAI Node (Langchain)  
  - *Role:* Uses GPT-4 to select the best keyword between the two top trends for SEO.  
  - *Configuration:* Prompt instructs the model to pick one keyword based on relevance and trendiness, returning only the selected keyword.  
  - *Input:* Output from "Extract Top 2 Trends".  
  - *Output:* Single keyword string.  
  - *Credentials:* OpenAI API (GPT-4).  
  - *Edge Cases:* API errors, rate limits, prompt failure, ambiguous keywords.

---

### 2.3 Content Research

**Overview:**  
Performs detailed research on the selected topic using the Perplexity AI API to obtain factual summaries from reputable sources.

**Nodes Involved:**  
- Research Reliable Sources  
- Convert Research Format

**Node Details:**

- **Research Reliable Sources**  
  - *Type:* HTTP Request  
  - *Role:* Posts a chat completion request to Perplexity API with the system prompt as a professional news researcher and user prompt to research the selected topic.  
  - *Input:* Selected topic from "Select Best SEO Topic".  
  - *Output:* JSON with research results including content and citations.  
  - *Credentials:* Generic HTTP Header Auth (Perplexity API key).  
  - *Edge Cases:* API throttling, invalid responses, network failures.

- **Convert Research Format**  
  - *Type:* Set Node  
  - *Role:* Processes Perplexity AI response to replace placeholders [1], [2], etc. with actual source citations for readability and hyperlinking.  
  - *Input:* From "Research Reliable Sources".  
  - *Output:* Formatted research string with integrated source links.  
  - *Edge Cases:* Missing citations, indexing errors.

---

### 2.4 Blog Drafting

**Overview:**  
Generates a detailed SEO-optimized blog post draft in Spanish using GPT-4 with the selected topic, optional keywords, and research findings.

**Nodes Involved:**  
- Draft Blog Content

**Node Details:**

- **Draft Blog Content**  
  - *Type:* OpenAI Node (Langchain)  
  - *Role:* Produces a blog post draft using a detailed prompt instructing to include the topic in title/H1, integrate research with source URLs, write engaging content suitable for a year 5 reading level, and produce 1500-2000 words minimum. Output is exclusively the blog content in Spanish.  
  - *Input:* Formatted research from "Convert Research Format", selected topic, and optional keywords.  
  - *Output:* Blog post text as string.  
  - *Credentials:* OpenAI API (GPT-4).  
  - *Edge Cases:* API limits, incomplete output, language output issues.

---

### 2.5 Internal Linking Preparation

**Overview:**  
Retrieves previously published blog posts from Google Sheets to prepare for internal link insertion.

**Nodes Involved:**  
- Find Previous Posts  
- Prepare Internal Link Dataset

**Node Details:**

- **Find Previous Posts**  
  - *Type:* Google Sheets  
  - *Role:* Reads all entries from a configured Google Sheets document serving as a database of previous blog posts.  
  - *Input:* From "Draft Blog Content" (triggered after draft generation).  
  - *Output:* Rows with prior blog post details (title, link, etc.).  
  - *Credentials:* Google Sheets OAuth2.  
  - *Edge Cases:* Permission errors, empty sheets, API quota.

- **Prepare Internal Link Dataset**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all retrieved items into a single array under the field "previous-posts" for easier access in later nodes.  
  - *Input:* From "Find Previous Posts".  
  - *Output:* Aggregated dataset of previous posts.  
  - *Edge Cases:* Empty input data.

---

### 2.6 Internal Link Insertion

**Overview:**  
Uses AI to analyze the new blog post draft and previous posts to insert at least five relevant internal links naturally into the content.

**Nodes Involved:**  
- Insert Internal SEO Links

**Node Details:**

- **Insert Internal SEO Links**  
  - *Type:* OpenAI Node (Langchain)  
  - *Role:* Given the new blog content and previous posts data, the AI adds internal linking URLs at relevant places in the new post without removing existing links or content. Output is the revised blog post text with link placements.  
  - *Input:* Aggregated previous posts and draft blog content.  
  - *Output:* Blog post content with internal links inserted.  
  - *Credentials:* OpenAI API (o1-mini model).  
  - *Edge Cases:* Insufficient relevant previous posts, AI misplacement of links.

---

### 2.7 SEO Metadata and Formatting

**Overview:**  
Generates semantic HTML code for the blog post, creates SEO-friendly slugs, blog titles, and meta descriptions using AI.

**Nodes Involved:**  
- Generate Semantic HTML  
- Create SEO Slug  
- Generate Blog Title  
- Create Meta Description

**Node Details:**

- **Generate Semantic HTML**  
  - *Type:* OpenAI Node (Langchain)  
  - *Role:* Converts the blog post text with internal links into well-structured HTML formatted according to a strict style guide (text color white, font Arial, blue hyperlinks, headings with blue underline, etc.).  
  - *Input:* Blog post content with internal links.  
  - *Output:* HTML code string suitable for WordPress post content.  
  - *Credentials:* OpenAI API (o1-preview model).  
  - *Edge Cases:* HTML formatting errors, truncated output.

- **Create SEO Slug**  
  - *Type:* OpenAI Node (Langchain)  
  - *Role:* Generates a concise URL slug (4-5 words max) including the primary keyword, suitable for SEO-friendly URLs.  
  - *Input:* Blog post content with internal links.  
  - *Output:* Slug string.  
  - *Credentials:* OpenAI API (gpt-4o-mini).  
  - *Edge Cases:* Ambiguous slug, keyword omission.

- **Generate Blog Title**  
  - *Type:* OpenAI Node (Langchain)  
  - *Role:* Extracts a suitable blog post title from the blog content, including the primary keyword, designed to inform users clearly.  
  - *Input:* Blog post content with internal links.  
  - *Output:* Title string.  
  - *Credentials:* OpenAI API (gpt-4o-2024-11-20).  
  - *Edge Cases:* Title truncation, irrelevant title.

- **Create Meta Description**  
  - *Type:* OpenAI Node (Langchain)  
  - *Role:* Creates a concise, keyword-rich meta description (150-160 characters) that accurately summarizes the blog post for SEO.  
  - *Input:* Blog post content with internal links.  
  - *Output:* Meta description string.  
  - *Credentials:* OpenAI API (gpt-4o-2024-11-20).  
  - *Edge Cases:* Over-length descriptions, keyword stuffing.

---

### 2.8 Cover Image Fetching

**Overview:**  
Fetches a relevant cover image URL from Google Images matching the selected topic.

**Nodes Involved:**  
- Fetch Cover Image  
- Store Cover Image URL

**Node Details:**

- **Fetch Cover Image**  
  - *Type:* HTTP Request  
  - *Role:* Queries SerpAPI’s Google Images engine with the selected blog topic to find relevant images.  
  - *Input:* Primary keyword from "Select Best SEO Topic".  
  - *Output:* JSON containing image search results.  
  - *Credentials:* SerpAPI HTTP Query Auth.  
  - *Edge Cases:* No images found, API errors.

- **Store Cover Image URL**  
  - *Type:* Set Node  
  - *Role:* Extracts the second image’s original URL from the fetch result and stores it in a variable for later use.  
  - *Input:* From "Fetch Cover Image".  
  - *Output:* JSON with `image-url` property.  
  - *Edge Cases:* Missing or malformed image results.

---

### 2.9 WordPress Publishing and Logging

**Overview:**  
Publishes the final blog post as a draft on WordPress with the generated title, slug, HTML content, and cover image, then logs publication details to Google Sheets.

**Nodes Involved:**  
- Publish to WordPress  
- Log Published Post

**Node Details:**

- **Publish to WordPress**  
  - *Type:* WordPress Node  
  - *Role:* Creates a new draft post with the given title, slug, featured image, author, category, and HTML content with internal links. Comments are disabled.  
  - *Input:* Title from "Generate Blog Title", slug from "Create SEO Slug", HTML from "Generate Semantic HTML", image URL from "Store Cover Image URL".  
  - *Output:* WordPress post object including post link and metadata.  
  - *Credentials:* WordPress OAuth2 or API Access.  
  - *Edge Cases:* Authentication failure, content size limits, media upload errors.

- **Log Published Post**  
  - *Type:* Google Sheets  
  - *Role:* Appends a new row to a Google Sheets document logging the post URL, title, transcript (empty), status, and HTML content for tracking and analytics.  
  - *Input:* Output from "Publish to WordPress".  
  - *Output:* Confirmation of data append.  
  - *Credentials:* Google Sheets OAuth2.  
  - *Edge Cases:* API quota limits, permission errors.

---

## 3. Summary Table

| Node Name                   | Node Type                      | Functional Role                                      | Input Node(s)             | Output Node(s)              | Sticky Note                                                                                                                  |
|-----------------------------|--------------------------------|-----------------------------------------------------|---------------------------|-----------------------------|------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger               | Initiates workflow on schedule                       | None                      | Get Google Trends            | See Sticky Note1: Pipeline overview and features.                                                                             |
| Get Google Trends           | HTTP Request                  | Fetches Google Trends data via SerpAPI               | Schedule Trigger          | Extract Top 2 Trends         | See Sticky Note1                                                                                                              |
| Extract Top 2 Trends        | Set                           | Extracts top 2 rising trends                         | Get Google Trends          | Filter High Volume Keywords  | See Sticky Note1                                                                                                              |
| Filter High Volume Keywords | Code (JavaScript)              | Filters keywords with volume > 30                    | Extract Top 2 Trends       | Select Best SEO Topic        | See Sticky Note1                                                                                                              |
| Select Best SEO Topic       | OpenAI (GPT-4)                | Chooses best SEO topic from top trends               | Filter High Volume Keywords| Research Reliable Sources    | See Sticky Note1                                                                                                              |
| Research Reliable Sources   | HTTP Request                  | Queries Perplexity API for research                   | Select Best SEO Topic      | Convert Research Format      | See Sticky Note1                                                                                                              |
| Convert Research Format     | Set                           | Formats research with source citations               | Research Reliable Sources  | Draft Blog Content           |                                                                                                                              |
| Draft Blog Content          | OpenAI (GPT-4)                | Generates detailed blog post draft                    | Convert Research Format    | Find Previous Posts          | See Sticky Note1                                                                                                              |
| Find Previous Posts         | Google Sheets                 | Retrieves previous blog post data                     | Draft Blog Content         | Prepare Internal Link Dataset|                                                                                                                              |
| Prepare Internal Link Dataset| Aggregate                    | Aggregates previous posts for internal linking       | Find Previous Posts        | Insert Internal SEO Links    |                                                                                                                              |
| Insert Internal SEO Links   | OpenAI (o1-mini)              | Inserts internal links into blog content              | Prepare Internal Link Dataset, Draft Blog Content | Generate Semantic HTML         |                                                                                                                              |
| Generate Semantic HTML      | OpenAI (o1-preview)           | Converts blog post text to styled HTML                | Insert Internal SEO Links  | Create SEO Slug              | See Sticky Note3: SEO optimization features.                                                                                 |
| Create SEO Slug             | OpenAI (gpt-4o-mini)          | Generates SEO-friendly URL slug                        | Generate Semantic HTML     | Generate Blog Title          | See Sticky Note3                                                                                                              |
| Generate Blog Title         | OpenAI (gpt-4o-2024-11-20)   | Extracts optimized blog title                          | Create SEO Slug            | Create Meta Description      | See Sticky Note3                                                                                                              |
| Create Meta Description     | OpenAI (gpt-4o-2024-11-20)   | Creates SEO meta description                           | Generate Blog Title        | Fetch Cover Image            | See Sticky Note3                                                                                                              |
| Fetch Cover Image           | HTTP Request                  | Retrieves relevant cover image from Google Images     | Create Meta Description    | Store Cover Image URL        |                                                                                                                              |
| Store Cover Image URL       | Set                           | Stores selected image URL for WordPress post          | Fetch Cover Image          | Publish to WordPress         |                                                                                                                              |
| Publish to WordPress        | WordPress                     | Publishes blog post draft with metadata and image    | Store Cover Image URL      | Log Published Post           | See Sticky Note2: Setup requirements; Google Sheets template link in Sticky Note4.                                            |
| Log Published Post          | Google Sheets                 | Logs published post details for tracking              | Publish to WordPress       | None                        | See Sticky Note4: Google Sheets template URL and columns information.                                                        |
| Sticky Note                 | Sticky Note                   | Documentation and instructions                        | None                      | None                        | See detailed content in the workflow overview section.                                                                       |
| Sticky Note1                | Sticky Note                   | Overview of complete pipeline and features           | None                      | None                        |                                                                                                                              |
| Sticky Note2                | Sticky Note                   | Setup requirements and API configuration details     | None                      | None                        |                                                                                                                              |
| Sticky Note3                | Sticky Note                   | SEO optimization features summary                     | None                      | None                        |                                                                                                                              |
| Sticky Note4                | Sticky Note                   | Google Sheets logging template and columns           | None                      | None                        | https://docs.google.com/spreadsheets/d/1ymP5AjpkCGSPrZF0W254lGMLVQ-0eHHCBL0Lz0sqilo/edit?usp=sharing                         |

---

## 4. Reproducing the Workflow from Scratch

Follow these steps in n8n to rebuild the workflow manually:

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configuration: Set interval (e.g., daily).  
   - Connect output to "Get Google Trends".

2. **Create Get Google Trends Node**  
   - Type: HTTP Request  
   - Method: GET  
   - URL: `https://serpapi.com/search?engine=google_trends`  
   - Authentication: HTTP Query Auth with SerpAPI credentials.  
   - Query Parameters:  
     - q = "ai agent"  
     - geo = "US"  
     - hl = "en"  
     - date = from 3 days ago to today (use expression: `{{$now.minus({ days: 3 }).format('yyyy-MM-dd')}} {{$now.format('yyyy-MM-dd')}}`)  
     - data_type = "RELATED_QUERIES"  
   - Connect output to "Extract Top 2 Trends".

3. **Create Extract Top 2 Trends Node**  
   - Type: Set  
   - Add JSON output mode with expression that extracts the first two rising queries and their extracted_value as score.  
   - Example structure:  
     ```
     {
       "most-trending": {
         "#1": {
           "query": "{{ $json.related_queries.rising[0].query }}",
           "score": "{{ $json.related_queries.rising[0].extracted_value }}"
         },
         "#2": {
           "query": "{{ $json.related_queries.rising[1].query }}",
           "score": "{{ $json.related_queries.rising[1].extracted_value }}"
         }
       }
     }
     ```  
   - Connect output to "Filter High Volume Keywords".

4. **Create Filter High Volume Keywords Node**  
   - Type: Code (JavaScript)  
   - Script: Filter `related_queries.top` for items with `extracted_value > 30` and output comma-separated queries.  
   - Connect output to "Select Best SEO Topic".

5. **Create Select Best SEO Topic Node**  
   - Type: OpenAI (Langchain)  
   - Model: GPT-4  
   - Prompt: Use detailed prompt to select best SEO keyword from the two top trends.  
   - Input variables: Pass JSON strings of the two keywords.  
   - Credentials: OpenAI API key configured.  
   - Connect output to "Research Reliable Sources".

6. **Create Research Reliable Sources Node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.perplexity.ai/chat/completions`  
   - Headers: Set authorization via HTTP Header Auth credentials.  
   - Body: JSON with model "sonar-pro", system and user messages requesting detailed research on selected topic.  
   - Connect output to "Convert Research Format".

7. **Create Convert Research Format Node**  
   - Type: Set  
   - Assign a field "research" replacing placeholders [1]..[10] with corresponding citations from the research JSON.  
   - Connect output to "Draft Blog Content".

8. **Create Draft Blog Content Node**  
   - Type: OpenAI (Langchain)  
   - Model: GPT-4  
   - Prompt: Detailed blog writing instructions in Spanish including topic, optional keywords, and research.  
   - Credentials: OpenAI API key.  
   - Connect output to "Find Previous Posts".

9. **Create Find Previous Posts Node**  
   - Type: Google Sheets  
   - Operation: Read rows from configured Google Sheet (CONTROL BLOG WEB).  
   - Credentials: Google Sheets OAuth2.  
   - Connect output to "Prepare Internal Link Dataset".

10. **Create Prepare Internal Link Dataset Node**  
    - Type: Aggregate  
    - Aggregate all rows into a single field "previous-posts".  
    - Connect output to "Insert Internal SEO Links".

11. **Create Insert Internal SEO Links Node**  
    - Type: OpenAI (Langchain)  
    - Model: o1-mini-2024-09-12  
    - Prompt: Insert at least 5 relevant internal links from previous posts into the current blog post without deleting existing content.  
    - Credentials: OpenAI API key.  
    - Connect output to "Generate Semantic HTML".

12. **Create Generate Semantic HTML Node**  
    - Type: OpenAI (Langchain)  
    - Model: o1-preview  
    - Prompt: Convert blog post with internal links into styled HTML formatted for WordPress, following strict inline styles and layout.  
    - Credentials: OpenAI API key.  
    - Connect output to "Create SEO Slug".

13. **Create Create SEO Slug Node**  
    - Type: OpenAI (Langchain)  
    - Model: gpt-4o-mini  
    - Prompt: Generate concise SEO slug including primary keyword (max 4-5 words).  
    - Credentials: OpenAI API key.  
    - Connect output to "Generate Blog Title".

14. **Create Generate Blog Title Node**  
    - Type: OpenAI (Langchain)  
    - Model: gpt-4o-2024-11-20  
    - Prompt: Extract blog post title including primary keyword from the blog content.  
    - Credentials: OpenAI API key.  
    - Connect output to "Create Meta Description".

15. **Create Create Meta Description Node**  
    - Type: OpenAI (Langchain)  
    - Model: gpt-4o-2024-11-20  
    - Prompt: Create concise, keyword-rich meta description (150-160 chars) summarizing blog content.  
    - Credentials: OpenAI API key.  
    - Connect output to "Fetch Cover Image".

16. **Create Fetch Cover Image Node**  
    - Type: HTTP Request  
    - Method: GET  
    - URL: `https://serpapi.com/search?engine=google_images`  
    - Authentication: HTTP Query Auth (SerpAPI)  
    - Query Parameters: q = selected topic, gl = "us"  
    - Connect output to "Store Cover Image URL".

17. **Create Store Cover Image URL Node**  
    - Type: Set  
    - Extract second image URL (`images_results[1].original`) and store as `image-url`.  
    - Connect output to "Publish to WordPress".

18. **Create Publish to WordPress Node**  
    - Type: WordPress  
    - Parameters:  
      - Title: From "Generate Blog Title"  
      - Slug: From "Create SEO Slug"  
      - Content: HTML with cover image `<img src="{{image-url}}" alt="Cover Image">` prepended  
      - Status: draft  
      - AuthorId, Categories set as per site  
      - Comment status: closed  
    - Credentials: WordPress API credentials.  
    - Connect output to "Log Published Post".

19. **Create Log Published Post Node**  
    - Type: Google Sheets  
    - Operation: Append row to logging sheet (VIDEO TRANSCRIPT PARA BLOG - LA TRIBU DIVISUAL)  
    - Columns: Link, Title, Transcription (empty), Status, HTML  
    - Credentials: Google Sheets OAuth2.

20. **Create Sticky Notes** (Optional, for documentation)  
    - Add sticky notes with overview, setup instructions, SEO features, and Google Sheets logging template.

---

## 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                  | Context or Link                                                                                     |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------|
| This workflow provides complete blog automation from trend detection to publication, eliminating manual content creation by using AI agents, Google Trends, Perplexity API, and WordPress integration.                                                                                                                                                                                       | Workflow purpose and overview (Sticky Note content)                                               |
| Setup requires API credentials for SerpAPI, Perplexity, OpenRouter/OpenAI, WordPress API access, and Google Sheets OAuth2 for logging.                                                                                                                                                                                                                                                     | Setup requirements (Sticky Note2)                                                                |
| SEO features include real-time trend monitoring, AI topic selection, semantic HTML generation, internal linking automation, meta description and slug creation, and publication logging.                                                                                                                                                                                                    | SEO optimization features (Sticky Note3)                                                        |
| Google Sheets template for post logging and internal link tracking is available at: https://docs.google.com/spreadsheets/d/1ymP5AjpkCGSPrZF0W254lGMLVQ-0eHHCBL0Lz0sqilo/edit?usp=sharing with columns for Date Published, Title, Slug, Target Keyword, WordPress URL, Internal Links Added | Google Sheets logging template (Sticky Note4)                                                    |
| LinkedIn profile and consulting links for workflow author Daniel Lianes: https://www.linkedin.com/in/daniel-lianes/ ; https://cal.com/averis/asesoria ; https://cal.com/averis/consultoria-personalizada                                                                                                                                                                                    | Author contact and consulting links (Sticky Note content)                                        |

---

**Disclaimer:**  
The text provided is exclusively derived from an n8n automated workflow. It strictly complies with content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.

---

This reference document enables users and AI agents to understand, reproduce, and customize the automated SEO blog post generation workflow end-to-end.