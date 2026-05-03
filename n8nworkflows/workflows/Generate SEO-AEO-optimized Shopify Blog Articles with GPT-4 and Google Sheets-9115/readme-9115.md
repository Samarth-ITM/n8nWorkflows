Generate SEO/AEO-optimized Shopify Blog Articles with GPT-4 and Google Sheets

https://n8nworkflows.xyz/workflows/generate-seo-aeo-optimized-shopify-blog-articles-with-gpt-4-and-google-sheets-9115


# Generate SEO/AEO-optimized Shopify Blog Articles with GPT-4 and Google Sheets

### 1. Workflow Overview

This n8n workflow automates the generation and publishing of SEO/AEO-optimized Shopify blog articles by leveraging GPT-4 and Google Sheets as input and tracking mechanisms. It is designed to streamline content creation based on keyword data, ensure unique slugs to avoid duplicates, generate hero images, and manage internal linking structures for SEO benefits.

The workflow is logically divided into the following blocks:

- **1.1 Input Configuration and Triggers:** Defines configuration parameters and triggers the workflow manually or on a schedule (Tuesdays and Fridays at 9 AM).

- **1.2 Data Retrieval and Normalization:** Reads keyword, links, and published article data from Google Sheets, normalizes this data, and merges it for processing.

- **1.3 Candidate Selection:** Selects the best keywords/clusters for article generation based on priority, volume, difficulty, and existing published content.

- **1.4 Existing Slugs Retrieval:** Queries Shopify via GraphQL to gather all existing blog article slugs to avoid duplication.

- **1.5 Content Generation with GPT-4:** Builds a prompt incorporating internal linking policies and requests GPT-4 to generate an SEO/AEO-optimized article in a strict JSON format.

- **1.6 Content Sanitization and Slug Finalization:** Parses GPT-4 output, cleans HTML content, and picks a unique slug avoiding conflicts with existing articles.

- **1.7 Hero Image Generation:** Requests an AI-generated hero image based on the article's image prompt.

- **1.8 Shopify Article Creation and SEO Metafields Update:** Creates the article on Shopify via REST API and updates SEO metafields via Shopify GraphQL API.

- **1.9 Google Sheets Update:** Appends new entries to "Published" and "Links" tabs to keep track of published articles and internal linking opportunities.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Configuration and Triggers

**Overview:** Sets up configuration parameters for the workflow and defines triggers to run the process manually or on a schedule.

**Nodes Involved:**
- Manual Trigger
- Schedule Trigger
- Set - Config
- Sticky Note (configuration instructions)
- Sticky Note (schedule info)

**Node Details:**

- **Manual Trigger**  
  - Type: Manual trigger node  
  - Role: Allows manual execution of workflow  
  - Configuration: Default, no parameters  
  - Connections: Triggers "Set - Config" node  
  - Potential Failures: None (manual activation)

- **Schedule Trigger**  
  - Type: Schedule trigger node  
  - Role: Runs workflow automatically on schedule  
  - Configuration: Cron expression set to run every Tuesday and Friday at 9 AM  
  - Connections: Triggers "Set - Config" node  
  - Potential Failures: Cron misconfiguration, timezone issues

- **Set - Config**  
  - Type: Set node  
  - Role: Central configuration storage for workflow parameters  
  - Configuration Parameters:  
    - maxPerRun: max number of articles per run (default 1)  
    - shopDomain: Shopify store domain (user must edit)  
    - siteBaseUrl: base URL for site (user must edit)  
    - blogId: Shopify blog ID (user must edit)  
    - blogHandle: blog handle slug (user must edit)  
    - tz: timezone (default Europe/Madrid)  
    - lang: language code (default en-EN)  
    - shopApiVersion: Shopify API version (default 2025-07)  
    - autoPublish: publish directly or draft (default false)  
    - sheetId: Google Sheet ID (user must edit)  
    - author: article author name (user must edit)  
  - Connections: Outputs to Google Sheets read nodes  
  - Edge Cases: Missing or incorrect configuration parameters will cause failures downstream

- **Sticky Note (CONFIG instructions)**  
  - Contains detailed instructions for required configuration variables and Google Sheets structure  
  - Reference for users to properly prepare environment before running workflow

- **Sticky Note (Schedule info)**  
  - Notes that the process runs on Tuesdays and Fridays at 9 AM automatically

---

#### 2.2 Data Retrieval and Normalization

**Overview:** Reads input data from Google Sheets ("Keywords", "Links", "Published" tabs) and normalizes to structured JSON for further processing.

**Nodes Involved:**
- Sheets - Read Keywords
- Sheets - Read Links
- Sheets - Read Published
- Merge
- Code - Normalize Inputs

**Node Details:**

- **Sheets - Read Keywords**  
  - Type: Google Sheets node  
  - Role: Reads keyword data tab containing keyword, cluster, intent, priority, volume, difficulty  
  - Configuration: Reads tab named "Keywords" from configured sheetId using Google Service Account credentials  
  - Edge Cases: Empty or malformed data rows; credentials issues

- **Sheets - Read Links**  
  - Type: Google Sheets node  
  - Role: Reads internal linking data with URLs and associated keywords  
  - Configuration: Reads tab named "Links" from configured sheetId  
  - Edge Cases: Missing URLs or keywords; empty rows

- **Sheets - Read Published**  
  - Type: Google Sheets node  
  - Role: Reads data of already published articles to avoid duplication  
  - Configuration: Reads tab named "Published" from configured sheetId  
  - Edge Cases: Missing data, inconsistent status flags

- **Merge**  
  - Type: Merge node  
  - Role: Combines keyword and published data streams for downstream processing  
  - Configuration: Default merge strategy  
  - Connections: Inputs from Sheets - Read Keywords and Sheets - Read Published

- **Code - Normalize Inputs**  
  - Type: Code node (JavaScript)  
  - Role:  
    - Normalizes and deduplicates keyword, published, and links data into structured objects  
    - Converts strings to ASCII, slugs keywords, builds lookup sets for existing slugs and published pairs  
  - Key Expressions: Implements slugification, normalization of cluster and keyword pairs  
  - Input: Merged Google Sheets rows  
  - Output: JSON with `keywords` array and `published` metadata object  
  - Edge Cases: Incorrect or missing columns, malformed rows, empty datasets

---

#### 2.3 Candidate Selection

**Overview:** Selects keywords for article generation based on priority, volume, difficulty, and filtering out already published keyword-cluster pairs.

**Nodes Involved:**
- Code - Pick Candidate
- Sticky Note (Pick a candidate)

**Node Details:**

- **Code - Pick Candidate**  
  - Type: Code node (JavaScript)  
  - Role:  
    - Filters out keywords that have already been published (based on pairs)  
    - Optionally avoids clusters already published (currently false)  
    - Sorts candidates by priority, volume (descending), difficulty (ascending), then alphabetically  
    - Picks up to maxPerRun keywords, ensuring one per cluster if enabled  
  - Key Expressions: Uses normalization functions for pair keys and cluster names  
  - Output: List of selected candidate keywords for article generation or a skip message if none eligible  
  - Edge Cases: No eligible keywords found, invalid or missing input data

- **Sticky Note (Pick a candidate)**  
  - Explains the logic focus on priority, volume, difficulty for keyword selection

---

#### 2.4 Existing Slugs Retrieval

**Overview:** Queries Shopify GraphQL API to retrieve all existing article slugs for the blog to prevent slug conflicts.

**Nodes Involved:**
- Code - Init Slug Parser
- Shopify - List Article Slugs (GraphQL)
- Code - Accumulate Slugs + Cursor
- If - More pages?
- Merge - Wiring
- Sticky Note (Slug list pipeline)

**Node Details:**

- **Code - Init Slug Parser**  
  - Type: Code node  
  - Role: Initializes state with empty existingSlugs array and null pagination cursor  
  - Output: JSON with `existingSlugs: []`, `__cursor: null`  
  - Edge Cases: None

- **Shopify - List Article Slugs**  
  - Type: HTTP Request node (GraphQL)  
  - Role: Calls Shopify GraphQL API to retrieve up to 250 blog articles per page (id and handle)  
  - Pagination: Uses `after` cursor for pagination  
  - Authentication: Shopify Admin Token (HTTP Header Auth) and Bearer Auth  
  - Output: JSON response with articles and pageInfo  
  - Edge Cases: API rate limits, auth errors, network timeouts

- **Code - Accumulate Slugs + Cursor**  
  - Type: Code node  
  - Role:  
    - Extracts article handles from API response edges  
    - Accumulates unique slugs in a list  
    - Updates pagination cursor and hasNext flag  
  - Output: Updated state with accumulated slugs and cursor for next page  
  - Edge Cases: Missing data, malformed response

- **If - More pages?**  
  - Type: If node  
  - Role: Checks if more pages of articles exist (`__hasNext === true`)  
  - True: Loops back to "Shopify - List Article Slugs" with updated cursor  
  - False: Proceeds to merge node to continue workflow  
  - Edge Cases: Infinite loop if flag stuck true

- **Merge - Wiring**  
  - Type: Merge node  
  - Role: Merges outputs from slug accumulation and other parts of the workflow for further processing

- **Sticky Note (Slug list pipeline)**  
  - Explains the purpose of gathering all used slugs to avoid duplicates

---

#### 2.5 Content Generation with GPT-4

**Overview:** Builds a detailed prompt based on the selected keyword, cluster, intent, internal links, and existing slugs, then requests GPT-4 to generate a structured SEO/AEO-optimized article.

**Nodes Involved:**
- Code - Build Prompt
- OpenAI - Chat Completions
- Sticky Note (OpenAI prompt context)

**Node Details:**

- **Code - Build Prompt**  
  - Type: Code node  
  - Role:  
    - Collects existing slugs, keyword data, and internal links  
    - Builds a system prompt describing the editorial and SEO/AEO constraints and style guide  
    - Includes an internal linking policy demanding exactly one relevant link insertion if applicable  
    - Defines strict JSON output format with fields like title, slug, seo_title, seo_description, tags, content_html, resumen, image prompts, and internal link metadata  
    - Enforces hard rules: word count, language style, HTML tag restrictions, slug uniqueness, internal link verification  
  - Output: JSON object containing messages (system + user) and response_format for GPT-4  
  - Edge Cases: Large or malformed link lists, missing keyword data

- **OpenAI - Chat Completions**  
  - Type: HTTP Request node  
  - Role: Sends chat completion request to OpenAI API with the constructed prompt  
  - Model: Uses "gpt-4o-mini" (GPT-4 optimized mini variant)  
  - Parameters: temperature 0.5 for balanced creativity  
  - Authentication: OpenAI API key via HTTP Header Auth  
  - Output: GPT-4 response with article JSON content  
  - Edge Cases: API rate limit, invalid API key, network errors, malformed responses

- **Sticky Note (Prepare the article with OpenAI)**  
  - Highlights this block's role in generating the article content and hero image prompt

---

#### 2.6 Content Sanitization and Slug Finalization

**Overview:** Parses GPT-4 output, cleans HTML, ensures single H1 tag, and picks a unique slug avoiding conflicts with existing Shopify articles.

**Nodes Involved:**
- Code - Sanitize + pick non-conflicting slug

**Node Details:**

- **Code - Sanitize + pick non-conflicting slug**  
  - Type: Code node  
  - Role:  
    - Parses GPT-4 JSON output safely, throws error on invalid JSON  
    - Sanitizes HTML content to keep only allowed tags (h1, h2, h3, p, ul, ol, li, a, strong, em, blockquote, code, pre)  
    - Ensures exactly one H1 tag at the start with the article title  
    - Generates slug candidates from GPT output slug or title and fallback backup slugs  
    - Ensures slug uniqueness by checking against existing slugs, adding numeric suffix if needed  
    - Cuts title, SEO title, and description to max allowed lengths  
    - Prepares final JSON object with sanitized fields for article creation  
  - Input: GPT-4 JSON, existing slugs list  
  - Output: Sanitized article data ready for Shopify API  
  - Edge Cases: Invalid or missing GPT response, slug conflicts, malformed HTML

---

#### 2.7 Hero Image Generation

**Overview:** Requests an AI-generated hero image for the article using OpenAI’s image generation API based on the image prompt from the content.

**Nodes Involved:**
- HTTP Request - OpenAI Images (Hero)

**Node Details:**

- **HTTP Request - OpenAI Images (Hero)**  
  - Type: HTTP Request node  
  - Role: Calls OpenAI images API to generate a 1536x1024 hero image from prompt  
  - Model: "gpt-image-1"  
  - Authentication: OpenAI API key via HTTP Header Auth  
  - Input: image_prompt string from sanitized article data  
  - Output: Base64-encoded image data embedded in JSON for Shopify article image attachment  
  - Edge Cases: API rate limits, invalid prompts, network issues

---

#### 2.8 Shopify Article Creation and SEO Metafields Update

**Overview:** Creates the article on Shopify via REST API, then updates SEO-related metafields for title and description using Shopify GraphQL mutation.

**Nodes Involved:**
- Shopify: Create Article (REST)
- Build Article GID (Code)
- Shopify: metafieldsSet (GraphQL)
- Sticky Note (Shopify article creation and SEO update)

**Node Details:**

- **Shopify: Create Article (REST)**  
  - Type: HTTP Request node  
  - Role: Posts new article JSON to Shopify REST API endpoint for blog articles  
  - URL constructed dynamically from config parameters (shopDomain, API version, blogId)  
  - Payload includes sanitized title, slug, author, tags, summary, content HTML, published flag, and base64 hero image attachment if present  
  - Authentication: Shopify Admin Token (HTTP Header Auth)  
  - Output: Shopify API response with created article data  
  - Edge Cases: API errors, invalid payload, auth failure

- **Build Article GID (Code)**  
  - Type: Code node  
  - Role: Extracts the Shopify article ID from response and builds the GraphQL global ID (gid) for metafield updates  
  - Throws error if article ID missing  
  - Output: JSON with articleId and articleGid  
  - Edge Cases: Missing article ID in response

- **Shopify: metafieldsSet (GraphQL)**  
  - Type: HTTP Request node (GraphQL)  
  - Role: Updates Shopify article metafields for SEO title and description tags  
  - Mutation with variables for metafields array specifying ownerId (articleGid), namespace "global", keys "title_tag" and "description_tag"  
  - Authentication: Shopify Admin Token  
  - Output: Mutation response with possible userErrors  
  - Edge Cases: GraphQL errors, invalid field values, auth errors

- **Sticky Note (Create Shopify article and update metafields)**  
  - Describes this block’s purpose in publishing article and SEO metadata update on Shopify

---

#### 2.9 Google Sheets Update

**Overview:** Updates the "Published" and "Links" tabs in Google Sheets to record the newly published article and update internal linking data for future articles.

**Nodes Involved:**
- Code - List of links for article
- Append row to "Published" tab (Google Sheets)
- Append row to "Links" tab (Google Sheets)
- Sticky Note (Google Sheets update)

**Node Details:**

- **Code - List of links for article**  
  - Type: Code node  
  - Role: Constructs an array of link objects from Google Sheets "Links" tab data with URL and keywords fields  
  - Output: JSON object with links array for prompt building  
  - Edge Cases: Empty rows, missing keywords or URLs

- **Append row to "Published" tab**  
  - Type: Google Sheets node  
  - Role: Appends a new row to "Published" tab with article metadata: datetime, keyword, cluster, title, slug, URL, status  
  - Uses data from Shopify article creation and candidate selection nodes  
  - Authentication: Google Service Account  
  - Edge Cases: API quota limits, invalid sheet ID, missing fields

- **Append row to "Links" tab**  
  - Type: Google Sheets node  
  - Role: Appends new internal linking entry with URL and keywords derived from newly created article  
  - Authentication: Google Service Account  
  - Edge Cases: Same as above

- **Sticky Note (Update to Google Sheets)**  
  - Explains that these updates track published articles and enhance internal linking data for SEO

---

### 3. Summary Table

| Node Name                        | Node Type                  | Functional Role                                   | Input Node(s)                             | Output Node(s)                          | Sticky Note                                               |
|---------------------------------|----------------------------|-------------------------------------------------|------------------------------------------|----------------------------------------|-----------------------------------------------------------|
| Manual Trigger                  | Manual Trigger             | Starts workflow manually                         | —                                        | Set - Config                          |                                                           |
| Schedule Trigger                | Schedule Trigger           | Scheduled automated trigger                      | —                                        | Set - Config                          | There is a cron to run the process each Tuesday and Friday at 9AM. |
| Set - Config                   | Set                        | Configuration parameters                         | Manual Trigger, Schedule Trigger          | Sheets - Read Keywords, Links, Published | CONFIG instructions about variables and Google Sheets structure |
| Sheets - Read Keywords          | Google Sheets              | Read keywords tab                                | Set - Config                            | Merge                                |                                                           |
| Sheets - Read Links             | Google Sheets              | Read internal links tab                          | Set - Config                            | Code - List of links for article      |                                                           |
| Sheets - Read Published         | Google Sheets              | Read published articles tab                      | Set - Config                            | Merge                                |                                                           |
| Merge                         | Merge                      | Merges keywords and published data               | Sheets - Read Keywords, Sheets - Read Published | Code - Normalize Inputs                |                                                           |
| Code - Normalize Inputs         | Code                       | Normalizes and structures input data             | Merge                                  | Code - Pick Candidate                 |                                                           |
| Code - Pick Candidate           | Code                       | Selects best keywords for article generation     | Code - Normalize Inputs                 | Code - Init Slug Parser, Merge - Wiring | Pick a candidate: explanation about priority-based selection |
| Code - Init Slug Parser         | Code                       | Initializes slug list and pagination cursor      | Code - Pick Candidate                   | Shopify - List Article Slugs           | Pipeline to make a list of already used slugs from Shopify blog |
| Shopify - List Article Slugs    | HTTP Request (GraphQL)     | Lists existing article slugs from Shopify blog   | Code - Init Slug Parser, If - More pages? | Code - Accumulate Slugs + Cursor       |                                                           |
| Code - Accumulate Slugs + Cursor| Code                       | Accumulates unique slugs and pagination cursor   | Shopify - List Article Slugs            | If - More pages?                     |                                                           |
| If - More pages?                | If                         | Checks if more pages exist for slug retrieval    | Code - Accumulate Slugs + Cursor        | Shopify - List Article Slugs, Merge - Wiring |                                                           |
| Merge - Wiring                 | Merge                      | Merges accumulated slugs with other data         | If - More pages?, Code - Pick Candidate, Code - List of links for article | Code - Build Prompt                   |                                                           |
| Code - List of links for article| Code                       | Formats internal links for prompt                 | Sheets - Read Links                     | Merge - Wiring                       |                                                           |
| Code - Build Prompt             | Code                       | Builds GPT-4 prompt including internal linking   | Merge - Wiring                         | OpenAI - Chat Completions             | Prepare the article with OpenAI to get the content and the hero image |
| OpenAI - Chat Completions       | HTTP Request               | Calls GPT-4 API to generate article content      | Code - Build Prompt                     | Code - Sanitize + pick non-conflicting slug |                                                           |
| Code - Sanitize + pick non-conflicting slug | Code              | Parses GPT output, sanitizes HTML, picks unique slug | OpenAI - Chat Completions               | HTTP Request - OpenAI Images (Hero)    |                                                           |
| HTTP Request - OpenAI Images (Hero) | HTTP Request           | Requests AI-generated hero image                  | Code - Sanitize + pick non-conflicting slug | Shopify: Create Article (REST)          |                                                           |
| Shopify: Create Article (REST)  | HTTP Request (REST)        | Creates article on Shopify                         | HTTP Request - OpenAI Images (Hero)     | Build Article GID                     | Create shopify article and update metafields for SEO     |
| Build Article GID               | Code                       | Builds Shopify article global ID                   | Shopify: Create Article (REST)          | Shopify: metafieldsSet (GraphQL)       |                                                           |
| Shopify: metafieldsSet (GraphQL)| HTTP Request (GraphQL)     | Updates SEO metafields for article                 | Build Article GID                      | Append row to "Links" tab, Append row to "Published" tab |                                                           |
| Append row to "Published" tab   | Google Sheets              | Updates published articles tracking                | Shopify: metafieldsSet                  | —                                    | Update to Google Sheets                                   |
| Append row to "Links" tab       | Google Sheets              | Updates internal linking data tracking             | Shopify: metafieldsSet                  | —                                    |                                                           |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Triggers**  
   - Add a **Manual Trigger** node for manual runs.  
   - Add a **Schedule Trigger** node with cron expression `0 9 * * 2,5` for Tuesdays and Fridays at 9 AM.

2. **Set Configuration**  
   - Add a **Set** node named "Set - Config" connected from both triggers.  
   - Add parameters:  
     - `maxPerRun` (number, default 1)  
     - `shopDomain` (string, your Shopify domain, e.g. your-store.myshopify.com)  
     - `siteBaseUrl` (string, your website base URL)  
     - `blogId` (string, your Shopify blog ID)  
     - `blogHandle` (string, the blog slug)  
     - `tz` (string, default "Europe/Madrid")  
     - `lang` (string, default "en-EN")  
     - `shopApiVersion` (string, default "2025-07")  
     - `autoPublish` (string, "false" for draft, "true" to publish)  
     - `sheetId` (string, your Google Sheet ID)  
     - `author` (string, article author name)

3. **Read Google Sheets Data**  
   - Add three **Google Sheets** nodes:  
     - "Sheets - Read Keywords": Read tab "Keywords" with service account credentials.  
     - "Sheets - Read Links": Read tab "Links".  
     - "Sheets - Read Published": Read tab "Published".  
   - Connect all three from "Set - Config".

4. **Merge Keywords and Published**  
   - Add a **Merge** node to combine "Sheets - Read Keywords" and "Sheets - Read Published" outputs.

5. **Normalize Inputs**  
   - Add a **Code** node "Code - Normalize Inputs" connected from the Merge node.  
   - Implement normalization logic: ASCII conversion, slug generation, deduplication, published pairs and clusters extraction.

6. **Pick Candidate Keywords**  
   - Add a **Code** node "Code - Pick Candidate" connected from "Code - Normalize Inputs".  
   - Implement filtering for unpublished keywords, sorting by priority/volume/difficulty, and picking maxPerRun with one per cluster.

7. **Initialize Slug Parsing**  
   - Add a **Code** node "Code - Init Slug Parser" connected from "Code - Pick Candidate".  
   - Initialize empty array for existing slugs and null cursor.

8. **Retrieve Existing Shopify Article Slugs**  
   - Add a **HTTP Request** node "Shopify - List Article Slugs" (GraphQL) connected from "Code - Init Slug Parser".  
   - Configure GraphQL query for blog articles with pagination.  
   - Use Shopify Admin Token credential.

9. **Accumulate Slugs and Pagination**  
   - Add a **Code** node "Code - Accumulate Slugs + Cursor" connected from "Shopify - List Article Slugs".  
   - Extract and accumulate slugs, update cursor and hasNext flag.

10. **Loop for Pagination**  
    - Add an **If** node "If - More pages?" checking if more pages exist.  
    - If true, loop back to "Shopify - List Article Slugs" with updated cursor.  
    - If false, proceed to next step.

11. **Merge Results for Prompt Construction**  
    - Add a **Merge** node "Merge - Wiring" with three inputs:  
      - From "If - More pages?" false branch  
      - From "Code - Pick Candidate" (second output)  
      - From "Sheets - Read Links" through a new **Code** node "Code - List of links for article" that formats links.

12. **Build GPT-4 Prompt**  
    - Add a **Code** node "Code - Build Prompt" connected from "Merge - Wiring".  
    - Implement prompt construction with system instructions, internal linking policy, user instructions, and strict JSON output format.

13. **Call OpenAI GPT-4 API**  
    - Add a **HTTP Request** node "OpenAI - Chat Completions" connected from "Code - Build Prompt".  
    - Configure with OpenAI API endpoint, model "gpt-4o-mini", temperature 0.5, and appropriate auth credentials.

14. **Sanitize GPT Output and Pick Slug**  
    - Add a **Code** node "Code - Sanitize + pick non-conflicting slug" connected from "OpenAI - Chat Completions".  
    - Parse JSON, sanitize HTML, enforce single H1, generate unique slug.

15. **Generate Hero Image**  
    - Add an **HTTP Request** node "HTTP Request - OpenAI Images (Hero)" connected from "Code - Sanitize + pick non-conflicting slug".  
    - Configure OpenAI images API with prompt, size 1536x1024, auth credentials.

16. **Create Shopify Article**  
    - Add an **HTTP Request** node "Shopify: Create Article (REST)" connected from "HTTP Request - OpenAI Images (Hero)".  
    - Configure REST POST to Shopify articles endpoint with article JSON body, including base64 image attachment.  
    - Use Shopify Admin Token for auth.

17. **Build Article GID**  
    - Add a **Code** node "Build Article GID" connected from "Shopify: Create Article (REST)".  
    - Extract article ID and build Shopify GID string.

18. **Update SEO Metafields on Shopify**  
    - Add an **HTTP Request** node "Shopify: metafieldsSet (GraphQL)" connected from "Build Article GID".  
    - Configure GraphQL mutation to set SEO title_tag and description_tag metafields.  
    - Use Shopify Admin Token for auth.

19. **Update Google Sheets Published and Links Tabs**  
    - Add two **Google Sheets** nodes connected from "Shopify: metafieldsSet (GraphQL)":  
      - "Append row to \"Published\" tab" to append metadata of published article.  
      - "Append row to \"Links\" tab" to add new internal link data.  
    - Use service account credentials and specify sheetId.

---

### 5. General Notes & Resources

| Note Content | Context or Link |
|--------------|-----------------|
| Google Sheets structure requires three tabs: Keywords, Links, and Published with specified columns for each. | Sticky Note near inputs |
| Configuration variables must be edited before use: Shopify domain, API version, blog ID, sheet ID, author, etc. | Sticky Note near Set - Config |
| Cron runs every Tuesday and Friday at 9 AM to automate article creation. | Sticky Note near Schedule Trigger |
| Internal linking policy enforces exactly one relevant internal link inserted naturally within the article content if applicable. | Prompt building code node comments |
| Article JSON output from GPT-4 must comply with strict validation rules: word count, slug uniqueness, character limits, HTML tags allowed, and internal links usage. | Prompt building code node comments |
| Shopify API calls use Admin Token with HTTP Header authentication; OpenAI calls require API keys via HTTP Header authentication. | Workflow node credentials |
| For internal linking, the "Links" tab is dynamically updated to improve link suggestions for future articles. | Google Sheets update block note |

---

This comprehensive documentation should enable efficient understanding, reproduction, and modification of the workflow, while anticipating possible errors like API rate limits, invalid data, or authentication failures.