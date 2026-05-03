Analyze LinkedIn top posts with Apify and OpenAI and log 7 ideas to Sheets

https://n8nworkflows.xyz/workflows/analyze-linkedin-top-posts-with-apify-and-openai-and-log-7-ideas-to-sheets-14778


# Analyze LinkedIn top posts with Apify and OpenAI and log 7 ideas to Sheets

# 1. Workflow Overview

This workflow analyzes high-performing LinkedIn posts from a target profile, extracts patterns from the strongest posts, generates 7 new post ideas with OpenAI, and appends the results to Google Sheets.

Its main use case is content research and ideation: a user provides a LinkedIn profile URL plus API credentials, the workflow collects posts through Apify, ranks them by impressions, asks OpenAI to derive insights and propose new posts, then logs the full output into a spreadsheet for review or publishing planning.

## 1.1 Input Reception and Validation

This block starts the workflow manually, loads runtime configuration values, validates that all required credentials and settings are present, and builds the payload expected by the Apify actor.

## 1.2 LinkedIn Post Collection and Ranking

This block calls Apify to retrieve LinkedIn posts, normalizes inconsistent response fields across actor outputs, filters unusable records, ranks posts by impressions, and prepares a compact analysis prompt from the top-performing examples.

## 1.3 AI Analysis and Post Generation

This block sends the ranked top posts to OpenAI using the Responses API with a strict JSON schema, then parses the model output into structured insights and exactly 7 generated post concepts.

## 1.4 Google Sheets Export Preparation

This block converts the AI output into one row per generated post, enriches each row with source-post context and metadata, and prepares the final column mapping expected by the spreadsheet.

## 1.5 Spreadsheet Logging

This block appends the generated rows into a selected Google Sheet tab.

---

# 2. Block-by-Block Analysis

## 2.1 Input Reception and Validation

### Overview
This block initializes the workflow and creates a validated runtime configuration object. It prevents downstream API calls from running with placeholder values or missing credentials.

### Nodes Involved
- Manual Trigger
- Set Config
- Validate Config + Build Actor Input

### Node Details

#### Manual Trigger
- **Type and technical role:** `n8n-nodes-base.manualTrigger`; manual entry point for testing or ad hoc execution.
- **Configuration choices:** No parameters configured. The workflow starts when executed manually in the editor.
- **Key expressions or variables used:** None.
- **Input and output connections:**
  - Input: none
  - Output: `Set Config`
- **Version-specific requirements:** Type version 1; standard manual trigger behavior.
- **Edge cases or potential failure types:**
  - No technical failure expected beyond normal n8n execution issues.
  - Since this is manual-only, the workflow does not run on a schedule unless another trigger is added.
- **Sub-workflow reference:** None.

#### Set Config
- **Type and technical role:** `n8n-nodes-base.set`; defines workflow configuration values in a single item.
- **Configuration choices:**
  - Creates these fields:
    - `apify_token`
    - `apify_actor_id`
    - `openai_api_key`
    - `linkedin_profile_url`
    - `results_limit`
  - Placeholder values are intentionally used for credentials and profile URL.
  - `results_limit` defaults to `20`.
- **Key expressions or variables used:** Static assignments only.
- **Input and output connections:**
  - Input: `Manual Trigger`
  - Output: `Validate Config + Build Actor Input`
- **Version-specific requirements:** Type version 3.4.
- **Edge cases or potential failure types:**
  - Not a failing node by itself, but placeholder values must be replaced.
  - Storing API keys directly in a Set node is functional but not ideal for production from a security standpoint.
- **Sub-workflow reference:** None.

#### Validate Config + Build Actor Input
- **Type and technical role:** `n8n-nodes-base.code`; validates required settings and constructs the request body for Apify.
- **Configuration choices:**
  - Reads the first input item as `config`.
  - Requires:
    - `apify_token`
    - `apify_actor_id`
    - `openai_api_key`
    - `linkedin_profile_url`
  - Treats empty values and placeholders beginning with `replace_with_your_` as missing.
  - Builds:
    - `actor_input.profileUrls` as an array containing the LinkedIn profile URL
    - `actor_input.resultsLimit` as a numeric value derived from `results_limit` or default `20`
- **Key expressions or variables used:**
  - `const config = $input.first().json;`
  - Placeholder detection with `String(value).startsWith('replace_with_your_')`
- **Input and output connections:**
  - Input: `Set Config`
  - Output: `Apify Get LinkedIn Posts`
- **Version-specific requirements:** Type version 2 for the Code node.
- **Edge cases or potential failure types:**
  - Throws an error if any required field is missing or still a placeholder.
  - `results_limit` is cast with `Number(...)`; invalid numeric input may become `NaN`, which is not explicitly guarded against.
  - Assumes the selected Apify actor accepts `profileUrls` and `resultsLimit` fields.
- **Sub-workflow reference:** None.

---

## 2.2 LinkedIn Post Collection and Ranking

### Overview
This block retrieves LinkedIn posts from Apify, normalizes varying field names in the returned dataset, ranks posts by impressions, and prepares a concise top-post summary for AI analysis.

### Nodes Involved
- Apify Get LinkedIn Posts
- Normalize + Rank LinkedIn Posts

### Node Details

#### Apify Get LinkedIn Posts
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls Apify’s actor endpoint and returns dataset items synchronously.
- **Configuration choices:**
  - Method: `POST`
  - URL is dynamically built as:
    - `https://api.apify.com/v2/acts/{actorId}/run-sync-get-dataset-items?...`
  - Query parameters embedded in URL:
    - `token={{$json.apify_token}}`
    - `clean=true`
    - `format=json`
  - Sends JSON body from `actor_input`
  - Uses synchronous dataset retrieval, so the workflow waits for actor completion
- **Key expressions or variables used:**
  - URL expression uses:
    - `$json.apify_actor_id`
    - `$json.apify_token`
  - Body expression:
    - `={{ $json.actor_input }}`
- **Input and output connections:**
  - Input: `Validate Config + Build Actor Input`
  - Output: `Normalize + Rank LinkedIn Posts`
- **Version-specific requirements:** HTTP Request node version 4.2.
- **Edge cases or potential failure types:**
  - Invalid Apify token or actor ID leads to 4xx errors.
  - Actor runtime delays or timeouts may affect execution.
  - If the actor input schema differs from `profileUrls/resultsLimit`, the actor may reject the request or return unexpected output.
  - Synchronous execution can fail on long actor runs depending on actor speed and HTTP limits.
- **Sub-workflow reference:** None.

#### Normalize + Rank LinkedIn Posts
- **Type and technical role:** `n8n-nodes-base.code`; converts heterogeneous Apify records into a consistent post model, filters invalid posts, ranks by impressions, and builds the OpenAI prompt.
- **Configuration choices:**
  - Reads all input items with `$input.all()`
  - Attempts to extract post text from many possible fields:
    - `text`, `postText`, `content`, `caption`, `description`, `fullText`, `body`, `linkedinText`, `postContent`
  - Attempts to extract impressions from:
    - `impressions`, `impressionCount`, `metrics.impressions`, `statistics.impressions`, `analytics.impressions`, `views`, `viewCount`, `engagementStats.impressions`
  - Also normalizes reactions, comments, reposts, post URL, publish date, and record ID
  - Filters posts to keep only entries with non-empty text and numeric impressions
  - Sorts descending by impressions
  - Assigns rank starting at 1
  - Keeps the top 5 posts for analysis
  - Builds a natural-language prompt asking OpenAI to:
    - identify patterns
    - summarize success factors
    - generate 7 original LinkedIn posts
    - return JSON only
- **Key expressions or variables used:**
  - Uses `$('Validate Config + Build Actor Input').first().json.linkedin_profile_url` to carry source profile metadata forward.
  - Constructs:
    - `top_posts`
    - `summary_for_prompt`
    - `analysis_prompt`
- **Input and output connections:**
  - Input: `Apify Get LinkedIn Posts`
  - Output: `OpenAI Analyze + Generate`
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - Throws an error if no posts with usable text and impression data are found.
  - If Apify returns impressions as formatted strings like `"12,345"` or `"12k"`, `Number(...)` may convert to `NaN`; such posts will be filtered out.
  - If the actor uses different field names not covered in the normalization logic, valid posts may be missed.
  - Only the top 5 posts are sent to AI even if more are available.
- **Sub-workflow reference:** None.

---

## 2.3 AI Analysis and Post Generation

### Overview
This block sends the top LinkedIn posts to OpenAI for structured analysis and post generation, then validates and reshapes the model response into workflow-ready JSON.

### Nodes Involved
- OpenAI Analyze + Generate
- Parse OpenAI Output

### Node Details

#### OpenAI Analyze + Generate
- **Type and technical role:** `n8n-nodes-base.httpRequest`; calls OpenAI’s Responses API directly instead of using a dedicated OpenAI node.
- **Configuration choices:**
  - Method: `POST`
  - URL: `https://api.openai.com/v1/responses`
  - Headers:
    - `Authorization: Bearer <openai_api_key>`
    - `Content-Type: application/json`
  - Request body includes:
    - `model: 'gpt-5.4-mini'`
    - `input`: system + user messages
    - `text.format.type: 'json_schema'`
    - strict JSON schema named `linkedin_content_engine_output`
  - Schema requires:
    - `insights`: array of strings, 3 to 10 items
    - `posts`: array of exactly 7 objects
  - Each generated post must include:
    - `title`
    - `hook`
    - `post_text`
    - `cta`
    - `format_type`
    - `why_it_should_work`
- **Key expressions or variables used:**
  - `{{$node["Set Config"].json["openai_api_key"]}}`
  - `$json.analysis_prompt`
- **Input and output connections:**
  - Input: `Normalize + Rank LinkedIn Posts`
  - Output: `Parse OpenAI Output`
- **Version-specific requirements:** HTTP Request node version 4.2.
- **Edge cases or potential failure types:**
  - Invalid API key causes authentication failure.
  - The chosen model name must exist and be available to the account.
  - API quota, rate limiting, or model access restrictions may cause errors.
  - If OpenAI returns a response shape different from the expected `output[0].content[0].text`, parsing may fail downstream.
  - Large prompts may hit token or payload limits if post texts are long.
- **Sub-workflow reference:** None.

#### Parse OpenAI Output
- **Type and technical role:** `n8n-nodes-base.code`; extracts the JSON text from the OpenAI response, parses it, and merges it with source metadata from the ranking stage.
- **Configuration choices:**
  - Reads response body from `$json`
  - Tries two possible text locations:
    - `response.output[0].content[0].text`
    - `response.output_text`
  - Parses the text with `JSON.parse`
  - Pulls source metadata from `Normalize + Rank LinkedIn Posts`
  - Returns:
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
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - Throws an error if no usable text is found in the OpenAI response.
  - Throws an error if returned text is not valid JSON.
  - Assumes `parsed.posts` exists and has a `.length`; malformed but parseable JSON could still break here.
  - If the upstream node name changes, the cross-node reference must be updated.
- **Sub-workflow reference:** None.

---

## 2.4 Google Sheets Export Preparation

### Overview
This block converts the parsed AI output into one item per generated post and maps source-post metrics plus insight fields into flat row objects suitable for spreadsheet insertion.

### Nodes Involved
- Prepare Google Sheets Rows

### Node Details

#### Prepare Google Sheets Rows
- **Type and technical role:** `n8n-nodes-base.code`; transforms a single structured result into multiple flat rows for Google Sheets.
- **Configuration choices:**
  - Reads:
    - `top_posts`
    - `insights`
    - `generated_posts`
  - Generates `generated_at` using the current ISO timestamp
  - Creates one output item per generated post
  - Associates each generated post with a source top post using index modulo:
    - `topPosts[index % Math.max(topPosts.length, 1)]`
  - Maps insight positions to fixed columns:
    - `insight_1` through `insight_4`
  - Maps generated post fields to:
    - `new_post_title`
    - `new_post_hook`
    - `new_post_text`
    - `new_post_cta`
    - `format_type`
    - `why_it_should_work`
- **Key expressions or variables used:**
  - `const data = $input.first().json;`
  - `new Date().toISOString()`
  - `index + 1` for `new_post_number`
- **Input and output connections:**
  - Input: `Parse OpenAI Output`
  - Output: `Append Rows to Google Sheet`
- **Version-specific requirements:** Code node version 2.
- **Edge cases or potential failure types:**
  - If `generated_posts` is empty or missing, the node returns an empty item list.
  - Only the first four insights are exported even though OpenAI may return up to 10.
  - Since there are 7 generated posts but at most 5 top posts, source posts will repeat via modulo assignment.
  - Missing post fields are converted to empty strings.
- **Sub-workflow reference:** None.

---

## 2.5 Spreadsheet Logging

### Overview
This block appends the prepared row items into a Google Sheet using a predefined column mapping.

### Nodes Involved
- Append Rows to Google Sheet

### Node Details

#### Append Rows to Google Sheet
- **Type and technical role:** `n8n-nodes-base.googleSheets`; appends rows to a Google Sheets worksheet.
- **Configuration choices:**
  - Operation: `append`
  - Spreadsheet selected by URL
  - Worksheet selected as `gid=0` / `Sheet1`
  - Mapping mode: define below
  - Explicit column mapping from workflow fields to sheet headers
  - Type conversion is disabled:
    - `attemptToConvertTypes: false`
    - `convertFieldsToString: false`
- **Key expressions or variables used:**
  - Examples:
    - `={{$json.generated_at}}`
    - `={{$json.source_profile}}`
    - `={{$json.new_post_title}}`
    - `={{$json.why_it_should_work}}`
- **Input and output connections:**
  - Input: `Prepare Google Sheets Rows`
  - Output: none
- **Version-specific requirements:** Google Sheets node version 4.7 with OAuth2 credential configured.
- **Edge cases or potential failure types:**
  - OAuth credential issues or expired access can prevent writes.
  - Sheet header names must match the configured mapping exactly.
  - Missing columns in the worksheet will cause append or mapping problems.
  - Writing mixed data types without conversion may produce unexpected cell formatting.
- **Sub-workflow reference:** None.

---

# 3. Summary Table

| Node Name | Node Type | Functional Role | Input Node(s) | Output Node(s) | Sticky Note |
|---|---|---|---|---|---|
| Manual Trigger | Manual Trigger | Manual workflow start |  | Set Config | ## Input and validation |
| Set Config | Set | Defines Apify, OpenAI, LinkedIn profile, and limit settings | Manual Trigger | Validate Config + Build Actor Input | ## Input and validation |
| Validate Config + Build Actor Input | Code | Validates required config and builds Apify actor input payload | Set Config | Apify Get LinkedIn Posts | ## Input and validation |
| Apify Get LinkedIn Posts | HTTP Request | Runs the Apify actor and fetches LinkedIn posts | Validate Config + Build Actor Input | Normalize + Rank LinkedIn Posts | ## Post collection and ranking |
| Normalize + Rank LinkedIn Posts | Code | Normalizes post fields, ranks by impressions, and builds AI prompt | Apify Get LinkedIn Posts | OpenAI Analyze + Generate | ## Post collection and ranking |
| OpenAI Analyze + Generate | HTTP Request | Sends top posts to OpenAI for structured insights and 7 new posts | Normalize + Rank LinkedIn Posts | Parse OpenAI Output | ## Analysis and generation |
| Parse OpenAI Output | Code | Extracts and parses the OpenAI JSON response | OpenAI Analyze + Generate | Prepare Google Sheets Rows | ## Analysis and generation |
| Prepare Google Sheets Rows | Code | Flattens generated content into one row per post for Sheets | Parse OpenAI Output | Append Rows to Google Sheet | ## Google Sheets export |
| Append Rows to Google Sheet | Google Sheets | Appends final rows to the target spreadsheet | Prepare Google Sheets Rows |  | ## Google Sheets export |
| Overview | Sticky Note | Setup and usage guidance for the whole workflow |  |  | # LinkedIn Top Posts to Google Sheets Content Engine  \n### HOW IT WORKS:  \nThis workflow helps you study what is already working on LinkedIn and turn those patterns into new post ideas.  \nIt pulls posts from a LinkedIn source through Apify, ranks them by impressions, sends the strongest examples to OpenAI for analysis, and writes the results to Google Sheets.  \n### HOW TO SET UP:  \nAdd your Apify token, LinkedIn actor ID, and OpenAI API key in the "Set Config" step.  \nReplace the example LinkedIn profile URL with the profile you want to analyze.  \nIn the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go.  \nBefore running the workflow, make sure your sheet already has these columns:  \nGenerated At  \nSource Profile  \nTop Post Rank  \nTop Post Text  \nTop Post Impressions  \nInsight 1  \nInsight 2  \nInsight 3  \nNew Post Title  \nNew Post Hook  \nNew Post Text  \nNew Post Cta  \nFormat Type  \nWhy It Should Work  \nTotal Posts Analyzed  \nTop Post Reactions  \nTop Post Comments  \nTop Post Reposts  \nInsight 4  \nNew Post Number  \nOnce that is in place, run the workflow and check the first output row to confirm everything is landing where you expect. |
| Input and Validation | Sticky Note | Visual label for the input block |  |  | ## Input and validation |
| Collection and Ranking | Sticky Note | Visual label for the ranking block |  |  | ## Post collection and ranking |
| Analysis and Generation | Sticky Note | Visual label for the AI block |  |  | ## Analysis and generation |
| Google Sheets Export | Sticky Note | Visual label for the export block |  |  | ## Google Sheets export |

---

# 4. Reproducing the Workflow from Scratch

1. **Create a new workflow**
   - In n8n, create a blank workflow.
   - Name it something like: `LinkedIn Top Posts to Google Sheets Content Engine`.

2. **Add a Manual Trigger node**
   - Node type: `Manual Trigger`
   - Leave default configuration.
   - This will be the workflow entry point.

3. **Add a Set node named `Set Config`**
   - Connect `Manual Trigger` → `Set Config`.
   - Add these fields:
     - `apify_token` as string
     - `apify_actor_id` as string
     - `openai_api_key` as string
     - `linkedin_profile_url` as string
     - `results_limit` as number
   - Example values:
     - `apify_token = replace_with_your_apify_token`
     - `apify_actor_id = replace_with_your_linkedin_actor_id`
     - `openai_api_key = replace_with_your_openai_api_key`
     - `linkedin_profile_url = https://www.linkedin.com/in/your-profile/`
     - `results_limit = 20`

4. **Add a Code node named `Validate Config + Build Actor Input`**
   - Connect `Set Config` → `Validate Config + Build Actor Input`.
   - Paste this logic conceptually:
     - Read the first input item.
     - Check that `apify_token`, `apify_actor_id`, `openai_api_key`, and `linkedin_profile_url` are present.
     - Reject empty values and placeholder values starting with `replace_with_your_`.
     - Build an `actor_input` object with:
       - `profileUrls: [linkedin_profile_url]`
       - `resultsLimit: Number(results_limit || 20)`
   - Output the original config plus `actor_input`.

5. **Add an HTTP Request node named `Apify Get LinkedIn Posts`**
   - Connect `Validate Config + Build Actor Input` → `Apify Get LinkedIn Posts`.
   - Configure:
     - Method: `POST`
     - URL:
       ```text
       =https://api.apify.com/v2/acts/{{$json.apify_actor_id}}/run-sync-get-dataset-items?token={{$json.apify_token}}&clean=true&format=json
       ```
     - Send Body: enabled
     - Body Content Type: JSON
     - JSON Body:
       ```text
       ={{ $json.actor_input }}
       ```
   - No additional auth is needed because the token is passed in the URL.
   - Ensure your chosen actor actually supports LinkedIn post extraction from `profileUrls`.

6. **Add a Code node named `Normalize + Rank LinkedIn Posts`**
   - Connect `Apify Get LinkedIn Posts` → `Normalize + Rank LinkedIn Posts`.
   - Implement logic to:
     - Read all returned items
     - Extract normalized fields for each post:
       - `text`
       - `impressions`
       - `reactions`
       - `comments`
       - `reposts`
       - `post_url`
       - `published_at`
       - `id`
     - Use fallback field names for inconsistent Apify output
     - Filter out posts without usable text or numeric impressions
     - Sort by impressions descending
     - Assign `rank`
     - Keep top 5 posts
     - Create `summary_for_prompt`
     - Create `analysis_prompt`
   - Also include:
     - `source_profile` from `linkedin_profile_url`
     - `total_posts_found`

7. **Add an HTTP Request node named `OpenAI Analyze + Generate`**
   - Connect `Normalize + Rank LinkedIn Posts` → `OpenAI Analyze + Generate`.
   - Configure:
     - Method: `POST`
     - URL: `https://api.openai.com/v1/responses`
     - Send Headers: enabled
     - Headers:
       - `Authorization` = `=Bearer {{$node["Set Config"].json["openai_api_key"]}}`
       - `Content-Type` = `application/json`
     - Send Body: enabled
     - Specify Body: JSON
   - Build the JSON body with:
     - `model: "gpt-5.4-mini"`
     - `input` containing:
       - a system message instructing the model to act as a LinkedIn strategist/copywriter
       - a user message set to `$json.analysis_prompt`
     - `text.format.type = "json_schema"`
     - `text.format.strict = true`
     - A schema requiring:
       - `insights`: array of strings, minimum 3, maximum 10
       - `posts`: array of exactly 7 objects
       - each post object with:
         - `title`
         - `hook`
         - `post_text`
         - `cta`
         - `format_type`
         - `why_it_should_work`
   - If your OpenAI account does not support this model name, replace it with a supported model while keeping the same schema-driven pattern.

8. **Add a Code node named `Parse OpenAI Output`**
   - Connect `OpenAI Analyze + Generate` → `Parse OpenAI Output`.
   - Implement logic to:
     - Read the API response
     - Extract generated text from:
       - `output[0].content[0].text`, or fallback to `output_text`
     - Throw an error if no text exists
     - Parse JSON from the text
     - Retrieve source data from `Normalize + Rank LinkedIn Posts`
     - Output:
       - `source_profile`
       - `total_posts_found`
       - `top_posts`
       - `insights`
       - `generated_posts`
       - `generated_count`

9. **Add a Code node named `Prepare Google Sheets Rows`**
   - Connect `Parse OpenAI Output` → `Prepare Google Sheets Rows`.
   - Implement logic to:
     - Read the first input item
     - Set `generated_at` to current ISO datetime
     - Loop through `generated_posts`
     - For each generated post, create one row object
     - Map each generated post to a source top post using modulo, so all 7 rows have source context even if only 5 top posts exist
     - Output fields:
       - `generated_at`
       - `source_profile`
       - `total_posts_analyzed`
       - `source_top_post_rank`
       - `source_top_post_text`
       - `source_top_post_impressions`
       - `source_top_post_reactions`
       - `source_top_post_comments`
       - `source_top_post_reposts`
       - `insight_1`
       - `insight_2`
       - `insight_3`
       - `insight_4`
       - `new_post_number`
       - `new_post_title`
       - `new_post_hook`
       - `new_post_text`
       - `new_post_cta`
       - `format_type`
       - `why_it_should_work`

10. **Prepare the target Google Sheet**
    - Create a spreadsheet and worksheet.
    - Add these header columns exactly:
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

11. **Create Google Sheets credentials**
    - In n8n, create or select a `Google Sheets OAuth2 API` credential.
    - Authenticate with the Google account that can edit the destination spreadsheet.

12. **Add a Google Sheets node named `Append Rows to Google Sheet`**
    - Connect `Prepare Google Sheets Rows` → `Append Rows to Google Sheet`.
    - Configure:
      - Resource: Spreadsheet / Rows
      - Operation: `Append`
      - Select the target spreadsheet
      - Select the worksheet/tab
      - Mapping mode: define below
    - Map fields as follows:
      - `Generated At` ← `{{$json.generated_at}}`
      - `Source Profile` ← `{{$json.source_profile}}`
      - `Top Post Rank` ← `{{$json.source_top_post_rank}}`
      - `Top Post Text` ← `{{$json.source_top_post_text}}`
      - `Top Post Impressions` ← `{{$json.source_top_post_impressions}}`
      - `Insight 1` ← `{{$json.insight_1}}`
      - `Insight 2` ← `{{$json.insight_2}}`
      - `Insight 3` ← `{{$json.insight_3}}`
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
      - `Insight 4` ← `{{$json.insight_4}}`
      - `New Post Number` ← `{{$json.new_post_number}}`
    - Keep append mode.
    - Type conversion can remain disabled as in the source workflow.

13. **Optionally add sticky notes for clarity**
    - Add note blocks titled:
      - `## Input and validation`
      - `## Post collection and ranking`
      - `## Analysis and generation`
      - `## Google Sheets export`
    - Add a larger overview note with setup instructions if desired.

14. **Replace placeholders with real values**
    - In `Set Config`, replace:
      - Apify token
      - Apify actor ID
      - OpenAI API key
      - LinkedIn profile URL
    - Confirm the selected Google Sheets credential is valid.

15. **Run the workflow manually**
    - Execute from `Manual Trigger`.
    - Verify:
      - Apify returns LinkedIn post items
      - ranking succeeds
      - OpenAI returns valid schema-compliant JSON
      - 7 rows are appended to the sheet

16. **Validate first-run output**
    - Check the first appended rows in Google Sheets.
    - Confirm that:
      - columns align correctly
      - top post metrics are populated
      - 7 generated posts were written
      - source profile and timestamp are correct

### Sub-workflow setup
This workflow does **not** use any Sub-Workflow / Execute Workflow node. No child workflow is required.

---

# 5. General Notes & Resources

| Note Content | Context or Link |
|---|---|
| LinkedIn Top Posts to Google Sheets Content Engine | Workflow title |
| This workflow helps you study what is already working on LinkedIn and turn those patterns into new post ideas. | Overview note |
| Add your Apify token, LinkedIn actor ID, and OpenAI API key in the `Set Config` step. | Setup guidance |
| Replace the example LinkedIn profile URL with the profile you want to analyze. | Setup guidance |
| In the Google Sheets node, choose the spreadsheet and worksheet where you want the output to go. | Setup guidance |
| Before running the workflow, make sure your sheet already has the required columns listed in the overview note. | Spreadsheet setup |
| Once that is in place, run the workflow and check the first output row to confirm everything is landing where you expect. | Validation guidance |

## Additional implementation notes
- The workflow uses direct HTTP requests for both Apify and OpenAI instead of dedicated nodes.
- The OpenAI step expects strict JSON output and will fail fast if the response cannot be parsed.
- The ranking logic depends heavily on the presence of impression data; actor outputs without impressions will be discarded.
- The Google Sheets export writes one row per generated post, so each execution should append exactly 7 rows if successful.