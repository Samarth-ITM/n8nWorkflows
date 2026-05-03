Repurpose YouTube Videos to Multiple Content Types with OpenRouter AI and Airtable

https://n8nworkflows.xyz/workflows/repurpose-youtube-videos-to-multiple-content-types-with-openrouter-ai-and-airtable-6899


# Repurpose YouTube Videos to Multiple Content Types with OpenRouter AI and Airtable

### 1. Workflow Overview

This workflow automates the repurposing of YouTube videos into multiple content types using OpenRouter AI and Airtable. It is designed for content creators, agencies, and businesses looking to scale content production by transforming video transcripts into structured summaries, tutorials, blog posts, newsletters, social media posts, and YouTube scripts—all managed within Airtable.

**Target Use Cases:**

- Automating content creation from YouTube videos without manual rewriting
- Generating diverse content formats optimized for different platforms
- Managing content lifecycle and status via Airtable
- Scaling video content repurposing for marketing, lead generation, and audience engagement

**Logical Blocks:**

- **1.1 Input Reception and Filtering:** Retrieve YouTube video URLs from Airtable and filter based on status.
- **1.2 Transcript Extraction:** Use Scrape Creator API to fetch video transcripts.
- **1.3 Transcript Processing:** Extract and clean transcript text for AI consumption.
- **1.4 Content Generation:** Use OpenRouter AI (GPT-4 Nano) with specialized prompt chains to generate various content types (summary, tutorial, blog post, LinkedIn post, newsletter, tweet, YouTube script).
- **1.5 Output Filtering and Mapping:** Filter AI outputs by content type and assemble matched data sets.
- **1.6 Data Upsert:** Update Airtable records with generated content and update status.
- **1.7 Record Management:** Delete records marked for deletion or no-operation.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Filtering

**Overview:**  
This block fetches YouTube video entries from Airtable and filters them based on their processing status: selected for processing, deletion, or ignore.

**Nodes Involved:**  
- Get YouTube URLs (Airtable)  
- Switch (Status-based routing)  
- Delete Selected Records (Airtable)  
- No Operation, do nothing (NoOp)

**Node Details:**

- **Get YouTube URLs**  
  - Type: Airtable node  
  - Role: Fetch all video entries from Airtable 'videos' table  
  - Config: Uses Airtable Personal Access Token credentials; pulls entire table  
  - Inputs: Manual Trigger start  
  - Outputs: List of video records with fields including URL, title, status, output types  
  - Failures: Airtable auth errors, rate limits, empty results

- **Switch**  
  - Type: Switch node  
  - Role: Routes records based on 'Status' field value  
  - Config:  
    - If Status = "selected" → process content generation  
    - If Status = "delete" → delete record  
    - Else → no operation  
  - Inputs: Output from Airtable  
  - Outputs: 3 branches with corresponding nodes  
  - Failures: Expression evaluation errors if 'Status' field missing or malformed

- **Delete Selected Records**  
  - Type: Airtable node  
  - Role: Deletes records marked for deletion by ID  
  - Config: Uses Airtable credentials; operation deleteRecord  
  - Inputs: From Switch 'delete' output  
  - Failures: Airtable permission errors, network failures

- **No Operation, do nothing**  
  - Type: NoOp node  
  - Role: Ends processing for records not selected or deleted  
  - Inputs: From Switch 'not selected' branch  
  - Outputs: None

---

#### 2.2 Transcript Extraction

**Overview:**  
For each selected video, this block calls the Scrape Creator API to extract the YouTube video transcript.

**Nodes Involved:**  
- Extract Data (Set)  
- Get Transcript (HTTP Request)  
- Extract Transcript+url+output+title (Set)

**Node Details:**

- **Extract Data**  
  - Type: Set node  
  - Role: Prepare data object with URL, output content types, and title extracted from Airtable record  
  - Config: Assigns 'url', 'output', 'title' fields from input JSON  
  - Inputs: From Switch 'selected' branch  
  - Outputs: Structured JSON for API request  
  - Failures: Missing fields in input JSON

- **Get Transcript**  
  - Type: HTTP Request node  
  - Role: Calls Scrape Creator API to get YouTube transcript for given video URL  
  - Config:  
    - Method: GET  
    - Query parameter: url = video URL from input  
    - Authentication: HTTP Header Auth using Scrape Creator API key  
  - Inputs: From Extract Data node  
  - Outputs: API response JSON with transcript data  
  - Failures: API quota exceeded, invalid URL, network timeouts

- **Extract Transcript+url+output+title**  
  - Type: Set node  
  - Role: Extract only transcript text and merge with existing metadata (url, output types, title) for downstream processing  
  - Config: Assigns 'text' from API response field 'transcript_only_text' and copies url, output, title  
  - Inputs: From Get Transcript  
  - Outputs: Cleaned transcript text and metadata  
  - Failures: Missing or malformed API response fields

---

#### 2.3 Content Generation

**Overview:**  
This block filters the requested output content types and uses specialized AI prompt chains to generate each content piece using OpenRouter GPT-4 Nano.

**Nodes Involved:**  
- Filter summary output (Filter)  
- Summary (Chain LLM with structured output parser)  
- Filter tutorial output (Filter)  
- tutorial (Chain LLM)  
- Filter blog post output (Filter)  
- blogpost (Chain LLM)  
- Filter linkedin output (Filter)  
- linkedin (Chain LLM)  
- Filter newsletter output (Filter)  
- newsletter (Chain LLM)  
- Filter tweeter output (Filter)  
- tweet (Chain LLM)  
- Filter youtube output (Filter)  
- youtube script (Chain LLM with structured output parser)  
- OpenRouter Chat Model (Language Model for all chain LLM nodes)

**Node Details:**

- **Filter nodes**  
  - Type: Filter nodes  
  - Role: Check if the 'output' array from input JSON includes relevant content type string (e.g., "summary", "tutorial")  
  - Config: Conditions check for array contains specific string  
  - Inputs: From Extract Transcript+url+output+title  
  - Outputs: Only pass forward if content requested  
  - Failures: Expression errors if output field missing or not array

- **Chain LLM nodes (Summary, tutorial, blogpost, linkedin, newsletter, tweet, youtube script)**  
  - Type: Langchain Chain LLM node  
  - Role: Generate the specific content piece by applying a detailed prompt to the transcript text  
  - Config:  
    - Model: OpenRouter GPT-4.1 Nano (via OpenRouter Chat Model node credentials)  
    - Prompt: Custom per content type, instructing AI to produce optimized content (e.g., tutorial with step-by-step instructions, blog post with business benefits, tweet with hashtags)  
    - Output parser: Some chains use Structured Output Parser nodes to enforce JSON format and structured content  
  - Inputs: Filtered transcript text  
  - Outputs: Generated content in string or JSON form  
  - Failures: API timeouts, malformed prompt outputs, parsing errors  
  - Notes: On error, set to continue (fail gracefully and produce partial output)

- **OpenRouter Chat Model**  
  - Type: Language Model node (GPT-4.1 Nano)  
  - Role: Provides AI text generation capabilities used by all Chain LLM nodes  
  - Config: Connected with valid OpenRouter API credentials  
  - Inputs: From each Chain LLM node's prompt  
  - Outputs: AI generated text responses  
  - Failures: Authentication errors, API quota exceeded, network issues

---

#### 2.4 Output Filtering and Mapping

**Overview:**  
This block collects all generated content pieces, matches them by URL, and constructs a unified data object for each video.

**Nodes Involved:**  
- Match Items (Code)

**Node Details:**

- **Match Items**  
  - Type: Code node (JavaScript)  
  - Role: Maps and merges multiple AI outputs (summary, tutorial, blog post, linkedin post, tweet, newsletter, youtube script) by video URL and title into a single JSON object per video  
  - Config:  
    - Extracts arrays from previous nodes by JSON paths  
    - Handles cases where some AI outputs may be missing (provides empty strings as fallback)  
    - Returns a consolidated array of objects with all content types per video  
  - Inputs: All Chain LLM outputs routed into this node  
  - Outputs: Array of merged content objects ready for Airtable update  
  - Failures: Index mismatches if nodes do not produce outputs for all items; null or undefined errors

---

#### 2.5 Data Upsert

**Overview:**  
Updates Airtable records with generated content and marks video status as processed.

**Nodes Involved:**  
- Update Airtable

**Node Details:**

- **Update Airtable**  
  - Type: Airtable node  
  - Role: Upserts (update or insert) video records by matching URL and fills fields with generated content  
  - Config:  
    - Uses Airtable Personal Access Token credentials  
    - Matches records by 'url' field  
    - Updates fields: Status = "processed", summary, tutorial, blog_post, linkedin, newsletter, tweet, and YouTube script fields (titles, hook, steps, conclusion)  
  - Inputs: From Match Items code node  
  - Outputs: Confirmation of update or insert  
  - Failures: Airtable API limits, field mapping errors, auth failures

---

### 3. Summary Table

| Node Name                      | Node Type                                 | Functional Role                                   | Input Node(s)                    | Output Node(s)                      | Sticky Note                                                                                                                     |
|--------------------------------|-------------------------------------------|--------------------------------------------------|---------------------------------|-----------------------------------|--------------------------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’   | Manual Trigger                            | Initiates the workflow manually                   | None                            | Get YouTube URLs                  |                                                                                                                                |
| Get YouTube URLs               | Airtable                                  | Fetch YouTube video records from Airtable        | When clicking ‘Test workflow’   | Switch                           |                                                                                                                                |
| Switch                        | Switch                                    | Routes records based on 'Status' field            | Get YouTube URLs                | Extract Data, Delete Selected Records, No Operation |                                                                                                                                |
| Delete Selected Records        | Airtable                                  | Deletes records marked for deletion                | Switch                         | None                            |                                                                                                                                |
| No Operation, do nothing       | NoOp                                      | Ends processing for non-selected or no-action records | Switch                         | None                            |                                                                                                                                |
| Extract Data                  | Set                                       | Prepares essential data fields (url, output, title) | Switch                         | Get Transcript                   |                                                                                                                                |
| Get Transcript                | HTTP Request                              | Calls Scrape Creator API to fetch YouTube transcript | Extract Data                   | Extract Transcript+url+output+title | ## Use Scrape Creator API https://docs.scrapecreators.com/v1/youtube/video/transcript                                              |
| Extract Transcript+url+output+title | Set                                  | Extracts clean transcript text and metadata       | Get Transcript                 | Filter summary output, Filter tutorial output, Filter blog post output, Filter summary output, Filter linkedin output, Filter newsletter output, Filter tweeter output, Filter youtube output |                                                                                                                                |
| Filter summary output          | Filter                                    | Passes items requesting summary content            | Extract Transcript+url+output+title | Summary                        | ## filter the outputs we want to process                                                                                       |
| Summary                      | Chain LLM (Langchain)                      | Generates structured summary and key takeaways    | Filter summary output          | Match Items                     |                                                                                                                                |
| Filter tutorial output         | Filter                                    | Passes items requesting tutorial content           | Extract Transcript+url+output+title | tutorial                      | ## filter the outputs we want to process                                                                                       |
| tutorial                     | Chain LLM (Langchain)                      | Generates comprehensive tutorial content           | Filter tutorial output         | Match Items                     |                                                                                                                                |
| Filter blog post output        | Filter                                    | Passes items requesting blog post content          | Extract Transcript+url+output+title | blogpost                     | ## filter the outputs we want to process                                                                                       |
| blogpost                    | Chain LLM (Langchain)                      | Generates conversion-optimized blog post content   | Filter blog post output        | Match Items                     |                                                                                                                                |
| Filter linkedin output         | Filter                                    | Passes items requesting LinkedIn post content      | Extract Transcript+url+output+title | linkedin                     | ## filter the outputs we want to process                                                                                       |
| linkedin                    | Chain LLM (Langchain)                      | Generates viral LinkedIn lead-generation post       | Filter linkedin output         | Match Items                     |                                                                                                                                |
| Filter newsletter output       | Filter                                    | Passes items requesting newsletter content         | Extract Transcript+url+output+title | newsletter                   | ## filter the outputs we want to process                                                                                       |
| newsletter                  | Chain LLM (Langchain)                      | Generates engaging newsletter content               | Filter newsletter output       | Match Items                     |                                                                                                                                |
| Filter tweeter output          | Filter                                    | Passes items requesting tweet content              | Extract Transcript+url+output+title | tweet                        | ## filter the outputs we want to process                                                                                       |
| tweet                       | Chain LLM (Langchain)                      | Generates promotional tweet content                 | Filter tweeter output          | Match Items                     |                                                                                                                                |
| Filter youtube output          | Filter                                    | Passes items requesting YouTube script content     | Extract Transcript+url+output+title | youtube script               | ## filter the outputs we want to process                                                                                       |
| youtube script              | Chain LLM (Langchain)                      | Generates structured YouTube video scripts          | Filter youtube output          | Match Items                     |                                                                                                                                |
| OpenRouter Chat Model          | Language Model (OpenRouter GPT-4.1 Nano) | Provides AI text generation for all chain nodes    | All Chain LLM nodes            | All Chain LLM nodes             |                                                                                                                                |
| Match Items                  | Code                                      | Matches and merges all generated content by URL    | Summary, tutorial, blogpost, linkedin, newsletter, tweet, youtube script | Update Airtable               |                                                                                                                                |
| Update Airtable               | Airtable                                  | Updates Airtable records with generated content    | Match Items                   | None                            |                                                                                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Node Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add Airtable Node to Fetch Videos**  
   - Node Type: Airtable  
   - Operation: Search/List all records in your Airtable base and table containing YouTube video URLs.  
   - Credentials: Airtable Personal Access Token with read access.  
   - Output: List of video records including URL, title, status, and output types.

3. **Add Switch Node to Filter Records by Status**  
   - Node Type: Switch  
   - Field: Use expression to check the 'Status' field in each record.  
   - Branches:  
     - If Status = "selected" → proceed with content generation.  
     - If Status = "delete" → delete record.  
     - Else → do nothing.

4. **Add Airtable Node to Delete Records Marked for Deletion**  
   - Node Type: Airtable  
   - Operation: Delete record by ID from 'delete' branch.  
   - Credentials: Same Airtable token with delete permissions.

5. **Add No Operation Node for Other Records**  
   - Node Type: NoOp  
   - Purpose: Ends processing for non-selected records.

6. **Add Set Node to Prepare Data for API**  
   - Node Type: Set  
   - Assign: url, output array, title fields from the input JSON.

7. **Add HTTP Request Node to Call YouTube Transcript API**  
   - Node Type: HTTP Request  
   - Method: GET  
   - URL: Scrape Creator API endpoint for YouTube transcript (e.g., https://api.scrapecreators.com/v1/youtube/video/transcript)  
   - Query Parameter: url = video URL from previous node  
   - Authentication: HTTP Header Auth with Scrape Creator API key.

8. **Add Set Node to Extract Transcript Text and Pass Metadata**  
   - Node Type: Set  
   - Extract 'transcript_only_text' from API response and assign to 'text'.  
   - Also pass url, output, title fields forward.

9. **Add Multiple Filter Nodes for Each Content Type**  
   - Node Type: Filter  
   - Condition: Check if the 'output' array contains each content type string ('summary', 'tutorial', 'blog-post', 'linkedin', 'newsletter', 'tweeter', 'youtube').  
   - Connect each filter output to respective content generation node.

10. **Add Chain LLM Nodes for Content Generation**  
    - Node Type: Langchain Chain LLM  
    - Model: Connect to an OpenRouter Chat Model node with GPT-4.1-nano.  
    - Prompt: Use custom prompts for each content type based on desired output (summary, tutorial, blog post, LinkedIn post, newsletter, tweet, YouTube script).  
    - Attach Structured Output Parsers for nodes requiring JSON output (Summary, YouTube script).  
    - Configure error handling to continue on failure.

11. **Add OpenRouter Chat Model Node**  
    - Node Type: Language Model  
    - Model: openai/gpt-4.1-nano (OpenRouter model)  
    - Credentials: OpenRouter API key configured.

12. **Add Code Node to Match and Merge Generated Contents**  
    - Node Type: Code  
    - Logic: Map and merge all generated content arrays by URL and title into single JSON objects per video.

13. **Add Airtable Node to Upsert Records**  
    - Node Type: Airtable  
    - Operation: Upsert record by matching 'url' field  
    - Fields to update: Status = "processed", summary, tutorial, blog_post, linkedin, newsletter, tweet, youtube_titles, youtube_hook, youtube_steps, youtube_conclusion, key_take_aways  
    - Credentials: Airtable Personal Access Token with write permissions.

14. **Connect Nodes Sequentially According to Data Flow:**  
    - Manual Trigger → Get YouTube URLs → Switch → (Selected) → Extract Data → Get Transcript → Extract Transcript+url+output+title → Filters → Chain LLMs → Match Items → Update Airtable  
    - Switch → Delete → Delete Selected Records  
    - Switch → Not selected → No Operation

15. **Set Up Credentials:**  
    - Airtable Personal Access Token (for all Airtable nodes)  
    - Scrape Creator API Key (for HTTP Request node)  
    - OpenRouter API Key (for OpenRouter Chat Model node)

16. **Validate and Test:**  
    - Run manual trigger on test YouTube URLs with status "selected".  
    - Confirm transcripts are fetched, content generated, and Airtable updated correctly.  
    - Handle errors gracefully and log issues.

---

### 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                               |
|-----------------------------------------------------------------------------------------------------------|---------------------------------------------------------------|
| Use Scrape Creator API for YouTube transcript extraction.                                                | https://docs.scrapecreators.com/v1/youtube/video/transcript   |
| Airtable Template for managing YouTube videos and content outputs.                                       | https://airtable.com/appOfTK0sDWqNiJyl/shrlNAbAT5bVQgJHb      |
| Workflow video demonstration on YouTube.                                                                | https://youtu.be/cbpMAyPNNlM?si=W8gn8TRjXSXpXC7G              |
| Prompts and chain design focus on content repurposing for marketing and lead generation.                 | Embedded in prompt parameters for Chain LLM nodes             |
| Use OpenRouter GPT-4.1 Nano model for cost-effective, high-quality AI generation.                        | Credentials node "OpenRouter account"                          |
| The workflow supports scalable content automation for creators, agencies, and businesses.                | Described in summary and tutorial content                      |
| Status field in Airtable controls workflow branching: "selected", "delete", "processed".                 | Important for filtering and record management                  |
| Output types field (array) determines which content pieces to generate per video.                        | Enables flexible content generation                            |
| Error handling set to continue on AI model failures to avoid stopping entire workflow.                   | "onError": "continueErrorOutput" in Chain LLM nodes           |
| Content output fields formatted for Airtable compatibility with bullet points and line breaks escaped.  | Ensures proper display in Airtable interface                   |

---

**Disclaimer:**  
The text provided originates exclusively from an automated workflow built with n8n, an integration and automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.