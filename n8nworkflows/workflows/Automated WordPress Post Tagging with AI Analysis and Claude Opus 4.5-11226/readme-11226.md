Automated WordPress Post Tagging with AI Analysis and Claude Opus 4.5

https://n8nworkflows.xyz/workflows/automated-wordpress-post-tagging-with-ai-analysis-and-claude-opus-4-5-11226


# Automated WordPress Post Tagging with AI Analysis and Claude Opus 4.5

### 1. Workflow Overview

This workflow automates the process of generating, creating, and assigning SEO-optimized tags to a specific WordPress blog post. It is designed for WordPress site managers and SEO specialists who want to enhance blog post discoverability by leveraging AI-driven content analysis and tag management.

The workflow logically divides into these blocks:

- **1.1 Input Initialization and Data Retrieval:** Receives manual trigger, sets input parameters (`post_id`, `url`), fetches the target post data and all existing tags from the WordPress site.

- **1.2 Data Preparation and Merging:** Cleans and merges post data with the full tag list to prepare for AI analysis.

- **1.3 AI Tag Analysis:** Uses an Anthropic Claude Opus 4.5 model acting as an SEO expert to analyze the post content and existing tags, selecting or generating up to four optimized tags.

- **1.4 Tag Processing:** Separates existing tags and new tags from AI output; creates new tags via WordPress API and collects their IDs.

- **1.5 Final Tag Aggregation and Post Update:** Merges all tag IDs (existing + newly created) and updates the original WordPress post with this optimized tag set.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization and Data Retrieval

- **Overview:**  
  This block starts the workflow manually, sets the target post ID and WordPress URL, then retrieves the blog post and all existing tags via the WordPress API.

- **Nodes Involved:**  
  - When clicking ‘Execute workflow’  
  - Set data  
  - Get article  
  - Get all tags  
  - Sanitaze Tags  
  - Merge

- **Node Details:**  

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to manually start the workflow.  
    - Configuration: No parameters; triggers on manual execution.  
    - Connections: Outputs to `Set data`.  
    - Edge Cases: None significant; user must manually trigger.

  - **Set data**  
    - Type: Set  
    - Role: Defines required input parameters `post_id` and `url`.  
    - Configuration: Hardcoded string values for `post_id` and WordPress site base `url`.  
    - Key expressions: None dynamic; uses static assignments.  
    - Connections: Outputs to `Get article`.  
    - Edge Cases: Must be correctly set before execution; wrong `post_id` or `url` causes downstream failures.

  - **Get article**  
    - Type: WordPress node (get operation)  
    - Role: Retrieves full blog post data for given `post_id` from WordPress.  
    - Configuration: Uses `postId = {{$json.post_id}}` from input.  
    - Credentials: Requires valid WordPress API credentials with read permissions.  
    - Outputs: Post JSON including title, content, excerpt, and tags.  
    - Connections: Outputs to `Get all tags` and `Merge`.  
    - Edge Cases: Post not found, network errors, or auth errors.

  - **Get all tags**  
    - Type: HTTP Request  
    - Role: Fetches all tags from WordPress REST API endpoint `/wp-json/wp/v2/tags`.  
    - Configuration: URL built dynamically from `url` set previously.  
    - Authentication: WordPress API credential reused.  
    - Outputs: Array of all existing tags with IDs and names.  
    - Connections: Outputs to `Sanitaze Tags`.  
    - Edge Cases: REST API failure, auth failure, empty tag list.

  - **Sanitaze Tags**  
    - Type: Aggregate  
    - Role: Extracts only `id` and `name` fields from raw tag data to standardize the tag list.  
    - Configuration: Includes specified fields `id` and `name`, aggregates all items.  
    - Outputs: Cleaned tag array as `tags` field.  
    - Connections: Outputs to `Merge`.  
    - Edge Cases: Empty input array, malformed data.

  - **Merge**  
    - Type: Merge (combine mode)  
    - Role: Combines post data from `Get article` with cleaned tag list from `Sanitaze Tags`.  
    - Configuration: Mode set to "combine all" to produce single merged item.  
    - Inputs: Post data and tag list.  
    - Outputs: Combined dataset for AI processing.  
    - Edge Cases: Mismatched inputs, empty arrays.

---

#### 2.2 AI Tag Analysis

- **Overview:**  
  This block sends the merged post content and tag list to an AI model (Claude Opus 4.5) that emulates an SEO expert. It analyzes the content’s SEO context and selects up to four existing tags or generates new tag suggestions.

- **Nodes Involved:**  
  - Anthropic Chat Model  
  - Tags Expert  
  - Structured Output Parser  
  - OpenAI Chat Model1 (present but not connected to main flow)

- **Node Details:**  

  - **Anthropic Chat Model**  
    - Type: LangChain Anthropic LLM Chat  
    - Role: AI model executing the SEO expert prompt using Claude Opus 4.5.  
    - Configuration: Model set to `claude-opus-4-5-20251101`.  
    - Credentials: Anthropic API key required.  
    - Outputs: Raw AI JSON response with tags and new_tags arrays.  
    - Connections: Outputs to `Tags Expert`.  
    - Edge Cases: API limits, timeouts, malformed responses.

  - **Tags Expert**  
    - Type: LangChain chain LLM with prompt  
    - Role: Prepares prompt text including post title, content, excerpt, and existing tags for AI analysis.  
    - Configuration: Prompt instructs AI to perform SEO content optimization and generate tag IDs and new tag names.  
    - Inputs: Merged post and tag data.  
    - Outputs: Parsed AI response with `tags` (existing tag IDs) and `new_tags` (new tag names).  
    - Connections: Outputs to `Existing Tags` and `New Tags`.  
    - Edge Cases: Expression errors in prompt, missing fields in input, AI response deviations.

  - **Structured Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Ensures AI response conforms to JSON schema with required fields `tags` (array of numbers) and `new_tags` (array of strings).  
    - Configuration: Auto-fix enabled; manual schema defined.  
    - Connections: Outputs parsed and validated data to `Tags Expert`.  
    - Edge Cases: Parsing failure, schema mismatch, incomplete AI output.

  - **OpenAI Chat Model1**  
    - Type: LangChain OpenAI LLM Chat  
    - Role: Present but unused in main flow; possibly for alternate AI model testing (GPT-4.1-mini).  
    - Credentials: OpenAI API key.  
    - Edge Cases: No direct role; can be removed or repurposed.

---

#### 2.3 Tag Processing

- **Overview:**  
  Processes AI output by separating existing tag IDs and new tag names. Existing tags are prepared for final update; new tags are created one by one via WordPress API calls.

- **Nodes Involved:**  
  - Existing Tags  
  - New Tags  
  - Split Out  
  - Loop Over Items  
  - Add tag  
  - Aggretate new tags  
  - Merge1

- **Node Details:**  

  - **Existing Tags**  
    - Type: Set  
    - Role: Extracts and sets existing tag IDs (`tags`) from AI output for later use.  
    - Configuration: Assigns `tags` = `{{$json.output.tags}}` (an array of numbers).  
    - Outputs: To `Merge1`.  
    - Edge Cases: Empty array, incorrect type.

  - **New Tags**  
    - Type: Set  
    - Role: Extracts new tag names (`new_tags`) from AI output for creation.  
    - Configuration: Assigns `new_tags` = `{{$json.output.new_tags}}` (an array of strings).  
    - Outputs: To `Split Out`.  
    - Edge Cases: Empty array means no new tags to create.

  - **Split Out**  
    - Type: Split Out  
    - Role: Splits array of new tag names into individual items for batch processing.  
    - Configuration: Field to split out: `new_tags`.  
    - Outputs: Each new tag name as separate item.  
    - Edge Cases: Empty input leads to no loop iterations.

  - **Loop Over Items**  
    - Type: Split In Batches (batch size = 1 by default)  
    - Role: Iterates over each new tag name for creation.  
    - Outputs: Parallel streams — one to create tags, one to aggregate results.  
    - Edge Cases: API errors during creation, throttling.

  - **Add tag**  
    - Type: HTTP Request (POST)  
    - Role: Creates a new tag in WordPress by posting to `/wp-json/wp/v2/tags` with tag name.  
    - Configuration: Body parameter `name` = current new tag string.  
    - Authentication: WordPress API credentials required with tag creation permissions.  
    - Outputs: Newly created tag data including ID.  
    - Edge Cases: Duplicate tag name error, auth issues, network failure.

  - **Aggretate new tags**  
    - Type: Aggregate  
    - Role: Collects all new tag IDs from the looped creation results.  
    - Configuration: Aggregates `id` field from each created tag into `tags` array.  
    - Outputs: To `Merge1`.  
    - Edge Cases: Empty aggregation if no new tags created.

  - **Merge1**  
    - Type: Merge  
    - Role: Combines existing tag IDs and newly created tag IDs into a single unified list.  
    - Configuration: Defaults to merging inputs to produce a single array.  
    - Outputs: To `Aggregate tags article`.  
    - Edge Cases: Empty inputs, duplicate tag IDs.

---

#### 2.4 Final Tag Aggregation and Post Update

- **Overview:**  
  Aggregates all tags into a clean list and updates the original post with the optimized tag set.

- **Nodes Involved:**  
  - Aggregate tags article  
  - Sanitize  
  - Update article

- **Node Details:**  

  - **Aggregate tags article**  
    - Type: Aggregate  
    - Role: Aggregates all tag IDs from merged data into one array.  
    - Configuration: Aggregates `tags` field.  
    - Outputs: To `Sanitize`.  
    - Edge Cases: Empty input array.

  - **Sanitize**  
    - Type: Code (JavaScript)  
    - Role: Cleans and deduplicates the array of tag IDs to ensure no duplicates before updating.  
    - Configuration: Custom JS code concatenates nested arrays and removes duplicates using Set.  
    - Outputs: Cleaned tag ID array as `tags`.  
    - Edge Cases: Unexpected data structure, empty input.

  - **Update article**  
    - Type: WordPress node (update operation)  
    - Role: Updates the target post’s tags with the final compiled list of tag IDs.  
    - Configuration: Uses `postId` from `Set data` and sets `tags` field with cleaned tag IDs.  
    - Credentials: WordPress API with write/update permissions.  
    - Edge Cases: Post not found, permission denied, network errors.

---

### 3. Summary Table

| Node Name                  | Node Type                             | Functional Role                                     | Input Node(s)                     | Output Node(s)                         | Sticky Note                                                                                                                                        |
|----------------------------|-------------------------------------|----------------------------------------------------|----------------------------------|--------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                      | Workflow entry point                               | -                                | Set data                             |                                                                                                                                                    |
| Set data                   | Set                                 | Defines input parameters `post_id` and `url`      | When clicking ‘Execute workflow’ | Get article                         |                                                                                                                                                    |
| Get article                | WordPress (get)                     | Fetches target post data                           | Set data                         | Get all tags, Merge                  |                                                                                                                                                    |
| Get all tags               | HTTP Request (GET)                  | Retrieves all existing tags from WordPress        | Get article                     | Sanitaze Tags                      |                                                                                                                                                    |
| Sanitaze Tags              | Aggregate                          | Extracts and cleans tag list (id, name)           | Get all tags                    | Merge                             |                                                                                                                                                    |
| Merge                     | Merge (combine all)                 | Combines post data and tag list                    | Get article, Sanitaze Tags       | Anthropic Chat Model                | Sticky Note1: ## STEP 1 - Get all tags from website                                                                                               |
| Anthropic Chat Model       | LangChain LLM Chat (Claude Opus 4.5) | AI SEO expert analysis for tag selection/generation | Merge                          | Tags Expert                       | Sticky Note2: ## STEP 2 - Tag Analysis                                                                                                            |
| Tags Expert                | LangChain Chain LLM                 | Prepares prompt, receives AI response              | Anthropic Chat Model, Structured Output Parser | Existing Tags, New Tags              | Sticky Note2: ## STEP 2 - Tag Analysis                                                                                                            |
| Structured Output Parser   | LangChain Output Parser             | Validates AI output format                          | OpenAI Chat Model1 (not used), Anthropic Chat Model | Tags Expert                      |                                                                                                                                                    |
| Existing Tags              | Set                                 | Extracts existing tag IDs from AI output           | Tags Expert                     | Merge1                            | Sticky Note3: ## STEP 3 - Existing Tags                                                                                                          |
| New Tags                   | Set                                 | Extracts new tag names from AI output               | Tags Expert                     | Split Out                        | Sticky Note4: ## STEP 4 - New Tags                                                                                                               |
| Split Out                  | Split Out                         | Splits new tag array into individual tag items    | New Tags                       | Loop Over Items                   | Sticky Note4: ## STEP 4 - New Tags                                                                                                               |
| Loop Over Items            | Split In Batches                  | Iterates over new tags to create each one          | Split Out                     | Aggretate new tags, Add tag       | Sticky Note4: ## STEP 4 - New Tags                                                                                                               |
| Add tag                    | HTTP Request (POST)                | Creates new tag via WordPress API                   | Loop Over Items                | Loop Over Items                   | Sticky Note4: ## STEP 4 - New Tags                                                                                                               |
| Aggretate new tags         | Aggregate                          | Aggregates IDs of newly created tags                | Loop Over Items                | Merge1                          | Sticky Note4: ## STEP 4 - New Tags                                                                                                               |
| Merge1                     | Merge                             | Combines existing and new tag ID arrays             | Existing Tags, Aggretate new tags | Aggregate tags article            | Sticky Note5: ## STEP 5 - Final Assignment                                                                                                       |
| Aggregate tags article     | Aggregate                          | Aggregates all tag IDs for final update             | Merge1                        | Sanitize                       | Sticky Note5: ## STEP 5 - Final Assignment                                                                                                       |
| Sanitize                   | Code                              | Cleans and deduplicates tag ID array                | Aggregate tags article         | Update article                | Sticky Note5: ## STEP 5 - Final Assignment                                                                                                       |
| Update article             | WordPress (update)                 | Updates the post’s tags with final optimized list   | Sanitize                      | -                              | Sticky Note5: ## STEP 5 - Final Assignment                                                                                                       |
| OpenAI Chat Model1         | LangChain LLM Chat (OpenAI GPT)   | Present but unused alternate AI model                | -                            | Structured Output Parser        |                                                                                                                                                    |
| Sticky Note                | Sticky Note                       | Documentation and overview                          | -                            | -                              | See notes in section 5                                                                                                                            |
| Sticky Note1               | Sticky Note                       | Step 1 explanation                                 | -                            | -                              | See notes in section 5                                                                                                                            |
| Sticky Note2               | Sticky Note                       | Step 2 explanation                                 | -                            | -                              | See notes in section 5                                                                                                                            |
| Sticky Note3               | Sticky Note                       | Step 3 explanation                                 | -                            | -                              | See notes in section 5                                                                                                                            |
| Sticky Note4               | Sticky Note                       | Step 4 explanation                                 | -                            | -                              | See notes in section 5                                                                                                                            |
| Sticky Note5               | Sticky Note                       | Step 5 explanation                                 | -                            | -                              | See notes in section 5                                                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger (`When clicking ‘Execute workflow’`)  
   - No parameters needed.

2. **Create Set Node to Define Inputs**  
   - Node Type: Set  
   - Assign two fields:  
     - `post_id` (string) — set to the WordPress post ID you want to tag.  
     - `url` (string) — set to your WordPress site base URL (e.g., `https://example.com/`).  
   - Connect Manual Trigger → Set.

3. **Create WordPress Node to Get Post**  
   - Node Type: WordPress  
   - Operation: Get  
   - Parameter: `postId` = `{{$json.post_id}}`  
   - Credentials: Configure WordPress API credentials with read access.  
   - Connect Set → Get article.

4. **Create HTTP Request Node to Get All Tags**  
   - Node Type: HTTP Request  
   - Method: GET  
   - URL: `={{ $json.url }}wp-json/wp/v2/tags`  
   - Authentication: Use the same WordPress credentials with tag read permissions.  
   - Connect Get article → Get all tags.

5. **Create Aggregate Node to Sanitize Tags**  
   - Node Type: Aggregate  
   - Operation: Aggregate all item data  
   - Include only fields: `id` and `name`  
   - Output field name: `tags`  
   - Connect Get all tags → Sanitaze Tags.

6. **Create Merge Node to Combine Post and Tags**  
   - Node Type: Merge  
   - Mode: Combine all inputs  
   - Connect Get article → Merge (input 1)  
   - Connect Sanitaze Tags → Merge (input 2).

7. **Create LangChain Anthropic Chat Model Node**  
   - Node Type: LangChain LLM Chat (Anthropic)  
   - Model: `claude-opus-4-5-20251101`  
   - Credentials: Anthropic API credentials with valid key.  
   - Connect Merge → Anthropic Chat Model.

8. **Create LangChain Chain LLM Node (Tags Expert)**  
   - Node Type: LangChain Chain LLM  
   - Prompt: Define detailed SEO expert instructions with placeholders for `title`, `content`, `excerpt`, and existing tags (use expressions).  
   - Output Parser: Enable structured output parser with JSON schema:  
     - `tags`: array of numbers  
     - `new_tags`: array of strings  
   - Connect Anthropic Chat Model → Tags Expert.

9. **Create Structured Output Parser Node**  
   - Node Type: LangChain Structured Output Parser  
   - Schema: Manual JSON schema matching AI output (see above).  
   - Auto-fix enabled.  
   - Connect OpenAI Chat Model1 (optional) and/or Anthropic Chat Model → Structured Output Parser → Tags Expert.

10. **Create Set Node for Existing Tags**  
    - Node Type: Set  
    - Assign field `tags` = `{{$json.output.tags}}` (array of tag IDs).  
    - Connect Tags Expert → Existing Tags.

11. **Create Set Node for New Tags**  
    - Node Type: Set  
    - Assign field `new_tags` = `{{$json.output.new_tags}}` (array of tag names).  
    - Connect Tags Expert → New Tags.

12. **Create Split Out Node**  
    - Node Type: Split Out  
    - Field to split: `new_tags`  
    - Connect New Tags → Split Out.

13. **Create Split In Batches Node (Loop Over Items)**  
    - Node Type: Split In Batches  
    - Batch size: default (1)  
    - Connect Split Out → Loop Over Items.

14. **Create HTTP Request Node to Add Tag**  
    - Node Type: HTTP Request (POST)  
    - URL: `={{ $json.url }}wp-json/wp/v2/tags`  
    - Body: JSON with `name` = current loop item (`{{$json.new_tags}}`)  
    - Authentication: WordPress credentials with tag creation rights.  
    - Connect Loop Over Items → Add tag.

15. **Create Aggregate Node to Collect New Tag IDs**  
    - Node Type: Aggregate  
    - Aggregate field: `id` from Add tag results  
    - Output field: `tags`  
    - Connect Loop Over Items → Aggretate new tags.

16. **Create Merge Node to Combine Existing and New Tags**  
    - Node Type: Merge  
    - Connect Existing Tags → Merge1 (input 1)  
    - Connect Aggretate new tags → Merge1 (input 2).

17. **Create Aggregate Node for Final Tag List**  
    - Node Type: Aggregate  
    - Aggregate field: `tags`  
    - Connect Merge1 → Aggregate tags article.

18. **Create Code Node to Sanitize Tags**  
    - Node Type: Code (JavaScript)  
    - Code: Concatenate nested arrays and remove duplicates using a Set.  
    - Connect Aggregate tags article → Sanitize.

19. **Create WordPress Node to Update Post**  
    - Node Type: WordPress  
    - Operation: Update  
    - `postId` = `{{$json.post_id}}`  
    - Update field: `tags` = `{{$json.tags}}` from sanitized output  
    - Credentials: WordPress API with write/update permissions.  
    - Connect Sanitize → Update article.

20. **(Optional) Add Sticky Notes**  
    - Add descriptive sticky notes near node groups to document each step.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                        | Context or Link                                                                                              |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| This workflow automates generating, creating, and assigning SEO-optimized WordPress tags using AI (Claude Opus 4.5) and WordPress REST API integration.                                                                                                           | Workflow overview sticky note                                                                                 |
| Step 1: Fetch target post and all existing tags from WordPress site to prepare data for AI analysis.                                                                                                                                                               | Sticky Note1                                                                                                  |
| Step 2: AI (Claude Opus 4.5) analyzes content as an SEO expert and selects or generates up to 4 relevant tags.                                                                                                                                                     | Sticky Note2                                                                                                  |
| Step 3: Prepares existing tag IDs for final update.                                                                                                                                                                                                                | Sticky Note3                                                                                                  |
| Step 4: Creates new tags from AI suggestions, one by one, via WordPress API.                                                                                                                                                                                       | Sticky Note4                                                                                                  |
| Step 5: Merges all tag IDs and updates the original post’s tags with the optimized list.                                                                                                                                                                           | Sticky Note5                                                                                                  |
| WordPress API credentials must have appropriate permissions for reading posts/tags and creating/updating tags.                                                                                                                                                     | Credential setup                                                                                              |
| AI API keys for Anthropic Claude Opus 4.5 are required and must be valid with sufficient quota.                                                                                                                                                                     | Credential setup                                                                                              |
| The workflow requires manual trigger execution with correctly set `post_id` and `url` fields to function.                                                                                                                                                          | User operation note                                                                                           |
| The LangChain nodes use structured output parsing to validate AI responses, reducing errors due to malformed outputs.                                                                                                                                              | AI output parsing best practice                                                                                |
| Potential failure points include API rate limits, WordPress REST API errors, missing permissions, malformed AI output, and network issues. Ensure proper error handling and monitoring when deploying in production.                                                  | Operational considerations                                                                                    |
| For deeper understanding of WordPress REST API endpoints and tag management, consult WordPress developer docs: https://developer.wordpress.org/rest-api/reference/tags/                                                                                              | External resource                                                                                             |
| For Anthropic Claude Opus 4.5 API usage and best practices, see: https://docs.anthropic.com/                                                                                                                                                                        | External resource                                                                                             |

---

**Disclaimer:** This documentation is based exclusively on an n8n workflow automation. It complies strictly with content policies and contains no illegal, offensive, or protected elements. All processed data is legal and public.