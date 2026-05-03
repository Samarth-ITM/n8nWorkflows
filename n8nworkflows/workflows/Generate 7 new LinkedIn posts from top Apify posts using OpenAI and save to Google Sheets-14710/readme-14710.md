Generate 7 new LinkedIn posts from top Apify posts using OpenAI and save to Google Sheets

https://n8nworkflows.xyz/workflows/generate-7-new-linkedin-posts-from-top-apify-posts-using-openai-and-save-to-google-sheets-14710


# Generate 7 new LinkedIn posts from top Apify posts using OpenAI and save to Google Sheets

# 1. Workflow Overview

This workflow collects LinkedIn posts for a target profile through Apify, ranks those posts by impressions, sends the strongest examples to OpenAI for pattern analysis and idea generation, then appends the generated content to Google Sheets.

Its main use case is content research and ideation: instead of manually studying high-performing LinkedIn posts, the workflow automates collection, extracts likely performance patterns, generates 7 new LinkedIn post ideas, and stores the results in a structured sheet for later review or publishing.

## 1.1 Input Reception and Configuration
The workflow starts manually, then loads a small configuration object containing:
- Apify token
- Apify actor ID
- OpenAI API key
- LinkedIn profile URL to analyze
- Results limit

It validates those values before any external API call is made.

## 1.2 Post Collection from Apify
Once validated, the workflow calls an Apify actor to fetch LinkedIn posts for the chosen profile. The returned dataset items become the raw source material for the rest of the workflow.

## 1.3 Post Normalization, Ranking, and Prompt Construction
The raw Apify output is normalized into a consistent post structure, ranked by impressions, and reduced to the top 5 posts. Those top posts are then summarized into a prompt for OpenAI.

## 1.4 OpenAI Analysis and Post Generation
The workflow sends the prompt to OpenAI using the Responses API and requests a strict JSON-schema output. OpenAI is asked to:
- identify winning patterns
- summarize insights
- generate exactly 7 new LinkedIn posts

## 1.5 Output Parsing and Google Sheets Export
The OpenAI response is parsed as JSON, combined with the source-post metadata, reshaped into one row per generated post, and appended into a Google Sheet.

---

# 2. Block-by-Block Analysis

## Block 1: Input and Validation

### Overview
This block initializes the workflow and prepares all required settings for downstream nodes. It ensures the workflow fails early if credentials or placeholders were not replaced.

### Nodes Involved
- Manual Trigger
- Set Config
- Validate Config + Build Actor Input

### Node Details

#### 1. Manual Trigger
- **Type and technical role:** `n8n-nodes-base.manualTrigger`  
  Entry-point node for manual execution inside n8n.
- **Configuration choices:**  
  No parameters are configured. It runs only when manually started.
- **Key expressions or variables used:**  
  None.
- **Input and output connections:**  
  - Input: none
  - Output: `Set Config`
- **Version-specific requirements:**  
  Type version `1`.
- **Edge cases or potential failure types:**  
  No technical failure expected, but the workflow only runs manually unless this node is replaced with another trigger.
- **Sub-workflow reference:**  
  None.

#### 2. Set Config
- **Type and technical role:** `n8n-nodes-base.set`  
  Creates the initial configuration payload used by the whole workflow.
- **Configuration choices:**  
  Defines five fields:
  - `apify_token` as string placeholder
  - `apify_actor_id` as string placeholder
  - `openai_api_key` as string placeholder
  - `linkedin_profile_url` as example LinkedIn profile URL
  - `results_limit` as numeric value, default `20`
- **Key expressions or variables used:**  
  Static assignments only.
- **Input and output connections:**  
  - Input: `Manual Trigger`
  - Output: `Validate Config + Build Actor Input`
- **Version-specific requirements:**  
  Type version `3.4`.
- **Edge cases or potential failure types:**  
  This node itself is unlikely to fail, but leaving placeholders unchanged will cause the next node to throw an error.
- **Sub-workflow reference:**  
  None.

#### 3. Validate Config + Build Actor Input
- **Type and technical role:** `n8n-nodes-base.code`  
  Validates required configuration fields and builds the request body expected by the Apify actor.
- **Configuration choices:**  
  The JavaScript code:
  - reads the first input item
  - checks required fields:
    - `apify_token`
    - `apify_actor_id`
    - `openai_api_key`
    - `linkedin_profile_url`
  - rejects missing, empty, null, undefined, or placeholder-like values starting with `replace_with_your_`
  - constructs `actor_input` with:
    - `profileUrls: [config.linkedin_profile_url]`
    - `resultsLimit: Number(config.results_limit || 20)`
- **Key expressions or variables used:**  
  - `config`
  - `requiredFields`
  - `missing`
  - `actor_input.profileUrls`
  - `actor_input.resultsLimit`
- **Input and output connections:**  
  - Input: `Set Config`
  - Output: `Apify Get LinkedIn Posts`
- **Version-specific requirements:**  
  Code node type version `2`.
- **Edge cases or potential failure types:**  
  - Throws an error if any required config field is missing or still a placeholder
  - `results_limit` is converted with `Number(...)`; non-numeric values may produce `NaN`, which may create downstream API issues
  - Invalid LinkedIn profile URLs are not fully validated here; Apify may reject or return empty results later
- **Sub-workflow reference:**  
  None.

---

## Block 2: Collection and Ranking

### Overview
This block fetches LinkedIn posts from Apify, normalizes potentially inconsistent response fields, ranks posts by impressions, and constructs the analysis prompt used by OpenAI.

### Nodes Involved
- Apify Get LinkedIn Posts
- Normalize + Rank LinkedIn Posts

### Node Details

#### 4. Apify Get LinkedIn Posts
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Sends a POST request to the Apify actor endpoint and retrieves dataset items synchronously.
- **Configuration choices:**  
  - Method: `POST`
  - URL dynamically built as:
    - `https://api.apify.com/v2/acts/{{$json.apify_actor_id}}/run-sync-get-dataset-items?token={{$json.apify_token}}&clean=true&format=json`
  - Request body: `{{$json.actor_input}}`
  - Body type: JSON
  - Response mode left as structured response object
- **Key expressions or variables used:**  
  - `$json.apify_actor_id`
  - `$json.apify_token`
  - `$json.actor_input`
- **Input and output connections:**  
  - Input: `Validate Config + Build Actor Input`
  - Output: `Normalize + Rank LinkedIn Posts`
- **Version-specific requirements:**  
  HTTP Request node type version `4.2`.
- **Edge cases or potential failure types:**  
  - Invalid Apify token
  - Invalid actor ID
  - Actor execution timeout or failure
  - Apify returning empty dataset items
  - Rate limiting or networking failures
  - Actor schema mismatch if the target actor expects additional parameters
- **Sub-workflow reference:**  
  None.

#### 5. Normalize + Rank LinkedIn Posts
- **Type and technical role:** `n8n-nodes-base.code`  
  Standardizes post fields from Apify output, sorts posts by impressions, selects top posts, and builds the OpenAI prompt.
- **Configuration choices:**  
  The JavaScript code:
  - processes all incoming items with `$input.all()`
  - extracts text from multiple possible fields:
    - `text`
    - `postText`
    - `content`
    - `caption`
    - `description`
    - `fullText`
    - `body`
    - `linkedinText`
    - `postContent`
  - extracts numeric metrics from multiple possible fields:
    - impressions
    - reactions
    - comments
    - reposts/shares
  - creates normalized objects with:
    - `rank`
    - `id`
    - `text`
    - `impressions`
    - `reactions`
    - `comments`
    - `reposts`
    - `post_url`
    - `published_at`
    - `raw`
  - filters out posts with empty text or invalid impression values
  - sorts descending by `impressions`
  - assigns `rank`
  - keeps top 5 posts
  - builds `summaryForPrompt`
  - creates `analysisPrompt`
  - outputs:
    - `source_profile`
    - `total_posts_found`
    - `top_posts`
    - `summary_for_prompt`
    - `analysis_prompt`
- **Key expressions or variables used:**  
  - `$input.all()`
  - `$('Validate Config + Build Actor Input').first().json.linkedin_profile_url`
  - `summaryForPrompt`
  - `analysisPrompt`
- **Input and output connections:**  
  - Input: `Apify Get LinkedIn Posts`
  - Output: `OpenAI Analyze + Generate`
- **Version-specific requirements:**  
  Code node type version `2`.
- **Edge cases or potential failure types:**  
  - Throws an error if no usable posts are found with text and numeric impression data
  - If Apify returns impressions as non-numeric strings, `Number(...)` may yield `NaN`
  - If the actor uses unexpected field names not covered by the normalization logic, valuable posts may be dropped
  - The workflow assumes impressions are the ranking metric; this may not fit all content strategies
  - Only top 5 posts are sent to OpenAI even if more are collected
- **Sub-workflow reference:**  
  None.

---

## Block 3: Analysis and Generation

### Overview
This block submits the ranked LinkedIn post summary to OpenAI and forces the result into a predictable JSON structure. It then parses that output and merges it with the source post context.

### Nodes Involved
- OpenAI Analyze + Generate
- Parse OpenAI Output

### Node Details

#### 6. OpenAI Analyze + Generate
- **Type and technical role:** `n8n-nodes-base.httpRequest`  
  Calls the OpenAI Responses API directly rather than using a dedicated OpenAI node.
- **Configuration choices:**  
  - Method: `POST`
  - URL: `https://api.openai.com/v1/responses`
  - Headers:
    - `Authorization: Bearer {{$node["Set Config"].json["openai_api_key"]}}`
    - `Content-Type: application/json`
  - JSON request body includes:
    - `model: 'gpt-5.4-mini'`
    - `input` with:
      - system instruction describing the assistant as a LinkedIn strategist/copywriter
      - user content from `$json.analysis_prompt`
    - `text.format.type: 'json_schema'`
    - `strict: true`
    - schema requiring:
      - `insights`: array of 3 to 10 strings
      - `posts`: array of exactly 7 objects
    - each post object must contain:
      - `title`
      - `hook`
      - `post_text`
      - `cta`
      - `format_type`
      - `why_it_should_work`
- **Key expressions or variables used:**  
  - `$json.analysis_prompt`
  - `$node["Set Config"].json["openai_api_key"]`
- **Input and output connections:**  
  - Input: `Normalize + Rank LinkedIn Posts`
  - Output: `Parse OpenAI Output`
- **Version-specific requirements:**  
  HTTP Request node type version `4.2`.  
  The workflow depends on OpenAI Responses API behavior and JSON-schema response formatting support. Model availability may vary by account or over time.
- **Edge cases or potential failure types:**  
  - Invalid or expired OpenAI API key
  - Model name unavailable for the account
  - API schema validation failure
  - Network or rate-limit errors
  - Output may still be missing expected fields despite strict schema if the API behavior changes
  - Prompt size could become an issue if post texts are very long
- **Sub-workflow reference:**  
  None.

#### 7. Parse OpenAI Output
- **Type and technical role:** `n8n-nodes-base.code`  
  Extracts text from the OpenAI response, parses it as JSON, and combines it with upstream source metadata.
- **Configuration choices:**  
  The JavaScript code:
  - reads the current response JSON
  - attempts to find model output in:
    - `response.output[0].content[0].text`
    - or fallback `response.output_text`
  - throws an error if no text is found
  - parses the raw text with `JSON.parse`
  - retrieves source data from `Normalize + Rank LinkedIn Posts`
  - outputs:
    - `source_profile`
    - `total_posts_found`
    - `top_posts`
    - `insights`
    - `generated_posts`
    - `generated_count`
- **Key expressions or variables used:**  
  - `response?.output?.[0]?.content?.[0]?.text`
  - `response?.output_text`
  - `$('Normalize + Rank LinkedIn Posts').first().json`
- **Input and output connections:**  
  - Input: `OpenAI Analyze + Generate`
  - Output: `Prepare Google Sheets Rows`
- **Version-specific requirements:**  
  Code node type version `2`.
- **Edge cases or potential failure types:**  
  - Throws if no usable text is found
  - Throws if returned text is not valid JSON
  - Throws if OpenAI changes the response envelope format
  - Assumes `parsed.posts.length` exists; malformed JSON would break this
- **Sub-workflow reference:**  
  None.

---

## Block 4: Google Sheets Export

### Overview
This block converts the parsed AI output into one spreadsheet row per generated post, then appends those rows into a preformatted Google Sheet.

### Nodes Involved
- Prepare Google Sheets Rows
- Append Rows to Google Sheet

### Node Details

#### 8. Prepare Google Sheets Rows
- **Type and technical role:** `n8n-nodes-base.code`  
  Reshapes the combined OpenAI output into multiple row items compatible with the Google Sheets append operation.
- **Configuration choices:**  
  The JavaScript code:
  - reads a single input object
  - extracts:
    - `top_posts`
    - `insights`
    - `generated_posts`
  - creates `generatedAt` as `new Date().toISOString()`
  - maps each generated post into a row object
  - assigns a source post by index using modulo:
    - `topPosts[index % Math.max(topPosts.length, 1)]`
  - outputs fields needed by the sheet, including:
    - generation timestamp
    - source profile
    - total analyzed posts
    - top post metrics
    - 4 insights
    - generated post number
    - title, hook, text, CTA
    - format type
    - why it should work
- **Key expressions or variables used:**  
  - `$input.first().json`
  - `new Date().toISOString()`
  - modulo mapping between generated posts and source top posts
- **Input and output connections:**  
  - Input: `Parse OpenAI Output`
  - Output: `Append Rows to Google Sheet`
- **Version-specific requirements:**  
  Code node type version `2`.
- **Edge cases or potential failure types:**  
  - If `generated_posts` is empty, no rows are produced
  - If fewer than 4 insights are returned, missing slots are filled with empty strings
  - Since 7 posts are generated and only up to 5 top posts exist, source top post references repeat cyclically
  - If upstream data shape changes, row fields may become blank
- **Sub-workflow reference:**  
  None.

#### 9. Append Rows to Google Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`  
  Appends prepared row items into a specific worksheet.
- **Configuration choices:**  
  - Operation: `append`
  - Google Sheets OAuth2 credentials required
  - Document ID points to a specific spreadsheet URL
  - Sheet name is selected as `gid=0` / `Sheet1`
  - Mapping mode: define columns manually
  - Explicit mapping between workflow fields and sheet columns:
    - `Generated At` ← `generated_at`
    - `Source Profile` ← `source_profile`
    - `Top Post Rank` ← `source_top_post_rank`
    - `Top Post Text` ← `source_top_post_text`
    - `Top Post Impressions` ← `source_top_post_impressions`
    - `Insight 1` ← `insight_1`
    - `Insight 2` ← `insight_2`
    - `Insight 3` ← `insight_3`
    - `Insight 4` ← `insight_4`
    - `New Post Title` ← `new_post_title`
    - `New Post Hook` ← `new_post_hook`
    - `New Post Text` ← `new_post_text`
    - `New Post Cta` ← `new_post_cta`
    - `Format Type` ← `format_type`
    - `Why It Should Work` ← `why_it_should_work`
    - `Total Posts Analyzed` ← `total_posts_analyzed`
    - `Top Post Reactions` ← `source_top_post_reactions`
    - `Top Post Comments` ← `source_top_post_comments`
    - `Top Post Reposts` ← `source_top_post_reposts`
    - `New Post Number` ← `new_post_number`
- **Key expressions or variables used:**  
  All mappings use expressions like `{{$json.field_name}}`.
- **Input and output connections:**  
  - Input: `Prepare Google Sheets Rows`
  - Output: none
- **Version-specific requirements:**  
  Google Sheets node type version `4.7`.
- **Edge cases or potential failure types:**  
  - OAuth credential failure or expired token
  - Spreadsheet or worksheet access denied
  - Column names in the sheet do not match the configured schema
  - Sheet may receive blank values if upstream fields are empty
  - Appending duplicate rows is possible if the workflow is rerun
- **Sub-workflow reference:**  
  None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Manual workflow entry point |  | Set Config | ## Input and validation |
| Set Config | Set | Defines tokens, actor ID, profile URL, OpenAI key, and result limit | Manual Trigger | Validate Config + Build Actor Input | ## Input and validation |
| Validate Config + Build Actor Input | Code | Validates config and builds Apify actor input | Set Config | Apify Get LinkedIn Posts | ## Input and validation |
| Apify Get LinkedIn Posts | HTTP Request | Calls Apify actor and fetches LinkedIn posts | Validate Config + Build Actor Input | Normalize + Rank LinkedIn Posts | ## Post collection and ranking |
| Normalize + Rank LinkedIn Posts | Code | Normalizes post fields, ranks by impressions, builds OpenAI prompt | Apify Get LinkedIn Posts | OpenAI Analyze + Generate | ## Post collection and ranking |
| OpenAI Analyze + Generate | HTTP Request | Sends ranked posts to OpenAI for insight extraction and post generation | Normalize + Rank LinkedIn Posts | Parse OpenAI Output | ## Analysis and generation |
| Parse OpenAI Output | Code | Parses OpenAI JSON and merges it with source metadata | OpenAI Analyze + Generate | Prepare Google Sheets Rows | ## Analysis and generation |
| Prepare Google Sheets Rows | Code | Converts generated posts into one row per spreadsheet entry | Parse OpenAI Output | Append Rows to Google Sheet | ## Google Sheets export |
| Append Rows to Google Sheet | Google Sheets | Appends output rows to the target spreadsheet | Prepare Google Sheets Rows |  | ## Google Sheets export |
| Overview | Sticky Note | Documentation note for setup and usage |  |  |  |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n**
   - Name it something like: `LinkedIn Top Posts to Google Sheets Content Engine`.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Leave default configuration.
   - This will be the workflow entry point.

3. **Add a Set node named `Set Config`**
   - Connect `Manual Trigger` → `Set Config`.
   - Add the following fields:
     - `apify_token` as String
     - `apify_actor_id` as String
     - `openai_api_key` as String
     - `linkedin_profile_url` as String
     - `results_limit` as Number
   - Suggested initial values:
     - `apify_token = replace_with_your_apify_token`
     - `apify_actor_id = replace_with_your_linkedin_actor_id`
     - `openai_api_key = replace_with_your_openai_api_key`
     - `linkedin_profile_url = https://www.linkedin.com/in/your-profile/`
     - `results_limit = 20`

4. **Add a Code node named `Validate Config + Build Actor Input`**
   - Connect `Set Config` → `Validate Config + Build Actor Input`.
   - Paste JavaScript logic that:
     - reads the first incoming item
     - checks required fields
     - throws an error if any field is missing or still starts with `replace_with_your_`
     - creates:
       - `actor_input.profileUrls = [linkedin_profile_url]`
       - `actor_input.resultsLimit = Number(results_limit || 20)`
   - Expected output shape should include all config fields plus:
     - `actor_input`

5. **Add an HTTP Request node named `Apify Get LinkedIn Posts`**
   - Connect `Validate Config + Build Actor Input` → `Apify Get LinkedIn Posts`.
   - Configure:
     - **Method:** `POST`
     - **URL:**  
       `https://api.apify.com/v2/acts/{{$json.apify_actor_id}}/run-sync-get-dataset-items?token={{$json.apify_token}}&clean=true&format=json`
     - **Send Body:** enabled
     - **Body Content Type / Specify Body:** JSON
     - **JSON Body:** `{{$json.actor_input}}`
   - This node does not use stored Apify credentials; the token is passed in the URL from the config object.

6. **Add a Code node named `Normalize + Rank LinkedIn Posts`**
   - Connect `Apify Get LinkedIn Posts` → `Normalize + Rank LinkedIn Posts`.
   - Add JavaScript that:
     - collects all items using `$input.all()`
     - normalizes possible text fields from Apify output
     - normalizes numeric metrics:
       - impressions
       - reactions
       - comments
       - reposts/shares
     - filters out posts without text or usable impression values
     - sorts descending by impressions
     - assigns rank numbers
     - keeps the top 5 posts
     - builds a JSON summary for prompt injection
     - creates a long `analysis_prompt`
   - Include a reference to the validated config node for the source profile:
     - `$('Validate Config + Build Actor Input').first().json.linkedin_profile_url`
   - Expected output fields:
     - `source_profile`
     - `total_posts_found`
     - `top_posts`
     - `summary_for_prompt`
     - `analysis_prompt`

7. **Add an HTTP Request node named `OpenAI Analyze + Generate`**
   - Connect `Normalize + Rank LinkedIn Posts` → `OpenAI Analyze + Generate`.
   - Configure:
     - **Method:** `POST`
     - **URL:** `https://api.openai.com/v1/responses`
     - **Send Headers:** enabled
     - **Headers:**
       - `Authorization` = `Bearer {{$node["Set Config"].json["openai_api_key"]}}`
       - `Content-Type` = `application/json`
     - **Send Body:** enabled
     - **Specify Body:** JSON
   - Use a JSON body that includes:
     - model name, here: `gpt-5.4-mini`
     - `input` array with:
       - system message describing the assistant as a LinkedIn content strategist/copywriter
       - user message from `{{$json.analysis_prompt}}`
     - `text.format.type = json_schema`
     - `strict = true`
     - a schema requiring:
       - `insights` as array of strings, minimum 3, maximum 10
       - `posts` as array with exactly 7 items
       - each post item containing:
         - `title`
         - `hook`
         - `post_text`
         - `cta`
         - `format_type`
         - `why_it_should_work`
   - Important: keep the schema strict so downstream parsing stays deterministic.

8. **Add a Code node named `Parse OpenAI Output`**
   - Connect `OpenAI Analyze + Generate` → `Parse OpenAI Output`.
   - Add JavaScript that:
     - reads the OpenAI HTTP response
     - extracts text from:
       - `response.output[0].content[0].text`
       - fallback to `response.output_text`
     - throws an error if no text exists
     - parses the result with `JSON.parse`
     - reads source data from:
       - `$('Normalize + Rank LinkedIn Posts').first().json`
     - outputs:
       - `source_profile`
       - `total_posts_found`
       - `top_posts`
       - `insights`
       - `generated_posts`
       - `generated_count`

9. **Add a Code node named `Prepare Google Sheets Rows`**
   - Connect `Parse OpenAI Output` → `Prepare Google Sheets Rows`.
   - Add JavaScript that:
     - reads the single combined object
     - extracts `top_posts`, `insights`, and `generated_posts`
     - sets `generated_at` to current ISO timestamp
     - maps each generated post into one n8n item
     - cycles through source top posts using modulo so every generated post gets a source reference
   - Each output item should include:
     - `generated_at`
     - `source_profile`
     - `total_posts_analyzed`
     - `source_top_post_rank`
     - `source_top_post_text`
     - `source_top_post_impressions`
     - `source_top_post_reactions`
     - `source_top_post_comments`
     - `source_top_post_reposts`
     - `insight_1` to `insight_4`
     - `new_post_number`
     - `new_post_title`
     - `new_post_hook`
     - `new_post_text`
     - `new_post_cta`
     - `format_type`
     - `why_it_should_work`

10. **Prepare the Google Sheet**
    - Create a Google Spreadsheet and worksheet before testing.
    - Add the following column headers exactly:
      - `Generated At`
      - `Source Profile`
      - `Top Post Rank`
      - `Top Post Text`
      - `Top Post Impressions`
      - `Insight 1`
      - `Insight 2`
      - `Insight 3`
      - `New Post Title`
      - `New Post Hook`
      - `New Post Text`
      - `New Post Cta`
      - `Format Type`
      - `Why It Should Work`
      - `Total Posts Analyzed`
      - `Top Post Reactions`
      - `Top Post Comments`
      - `Top Post Reposts`
      - `Insight 4`
      - `New Post Number`

11. **Add a Google Sheets node named `Append Rows to Google Sheet`**
    - Connect `Prepare Google Sheets Rows` → `Append Rows to Google Sheet`.
    - Configure:
      - **Resource/Operation:** Append rows
      - **Authentication:** Google Sheets OAuth2
      - Select or create Google Sheets OAuth2 credentials
      - Choose the target spreadsheet
      - Choose the correct worksheet
      - Use manual field mapping / define-below mode
    - Map fields as follows:
      - `Generated At` ← `{{$json.generated_at}}`
      - `Source Profile` ← `{{$json.source_profile}}`
      - `Top Post Rank` ← `{{$json.source_top_post_rank}}`
      - `Top Post Text` ← `{{$json.source_top_post_text}}`
      - `Top Post Impressions` ← `{{$json.source_top_post_impressions}}`
      - `Insight 1` ← `{{$json.insight_1}}`
      - `Insight 2` ← `{{$json.insight_2}}`
      - `Insight 3` ← `{{$json.insight_3}}`
      - `Insight 4` ← `{{$json.insight_4}}`
      - `New Post Title` ← `{{$json.new_post_title}}`
      - `New Post Hook` ← `{{$json.new_post_hook}}`
      - `New Post Text` ← `{{$json.new_post_text}}`
      - `New Post Cta` ← `{{$json.new_post_cta}}`
      - `Format Type` ← `{{$json.format_type}}`
      - `Why It Should Work` ← `{{$json.why_it_should_work}}`
      - `Total Posts Analyzed` ← `{{$json.total_posts_analyzed}}`
      - `Top Post Reactions` ← `{{$json.source_top_post_reactions}}`
      - `Top Post Comments` ← `{{$json.source_top_post_comments}}`
      - `Top Post Reposts` ← `{{$json.source_top_post_reposts}}`
      - `New Post Number` ← `{{$json.new_post_number}}`

12. **Create Google Sheets credentials**
    - In n8n, create `Google Sheets OAuth2 API` credentials.
    - Authenticate with a Google account that has edit access to the spreadsheet.
    - Reopen the node and select the spreadsheet and worksheet.

13. **Replace placeholders in `Set Config`**
    - Enter:
      - valid Apify token
      - valid LinkedIn actor ID
      - valid OpenAI API key
      - the LinkedIn profile URL you want to analyze
      - an appropriate results limit, such as 20

14. **Optionally add sticky notes for documentation**
    - Add notes matching the original organization:
      - `## Input and validation`
      - `## Post collection and ranking`
      - `## Analysis and generation`
      - `## Google Sheets export`
      - one overview note with setup guidance

15. **Test the workflow**
    - Execute from `Manual Trigger`.
    - Verify:
      - Apify returns posts
      - normalization produces ranked posts
      - OpenAI returns strict JSON
      - the Google Sheet receives 7 appended rows

16. **Validate output quality**
    - Check the first appended row in Sheets.
    - Confirm:
      - headers align correctly
      - text is not truncated unexpectedly
      - impressions and engagement metrics are populated
      - 7 generated posts are inserted

### Credential and integration expectations
- **Apify:** token is injected manually from the Set node; no dedicated credential object is used in this workflow.
- **OpenAI:** API key is also injected manually from the Set node; no dedicated credential object is used in this workflow.
- **Google Sheets:** requires OAuth2 credentials configured in n8n.

### No sub-workflows
This workflow does not call any sub-workflow and does not expose itself as one. It has a single entry point: `Manual Trigger`.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| LinkedIn Top Posts to Google Sheets Content Engine | Workflow title |
| This workflow helps you study what is already working on LinkedIn and turn those patterns into new post ideas. | Overview note |
| It pulls posts from a LinkedIn source through Apify, ranks them by impressions, sends the strongest examples to OpenAI for analysis, and writes the results to Google Sheets. | Overview note |
| Add your Apify token, LinkedIn actor ID, and OpenAI API key in the "Set Config" step. | Setup note |
| Replace the example LinkedIn profile URL with the profile you want to analyze. | Setup note |
| In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go. | Setup note |
| Before running the workflow, make sure your sheet already has these columns: Generated At; Source Profile; Top Post Rank; Top Post Text; Top Post Impressions; Insight 1; Insight 2; Insight 3; New Post Title; New Post Hook; New Post Text; New Post Cta; Format Type; Why It Should Work; Total Posts Analyzed; Top Post Reactions; Top Post Comments; Top Post Reposts; Insight 4; New Post Number | Sheet preparation note |
| Once that is in place, run the workflow and check the first output row to confirm everything is landing where you expect. | Validation note |