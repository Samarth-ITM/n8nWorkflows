Bulk SEO Content Generation with GPT, Gemini Images, and Webflow Draft Creation

https://n8nworkflows.xyz/workflows/bulk-seo-content-generation-with-gpt--gemini-images--and-webflow-draft-creation-9191


# Bulk SEO Content Generation with GPT, Gemini Images, and Webflow Draft Creation

---

### 1. Workflow Overview

This workflow automates the bulk generation of SEO-optimized blog content, including AI-generated featured images, and creates draft posts in a Webflow CMS collection. It is designed to streamline content production for websites targeting multiple long-tail keywords by integrating AI content creation, image generation, quality control, and CMS publishing.

**Target Use Cases:**
- Content marketing teams needing to mass-produce SEO articles.
- Agencies managing multiple keyword-driven content campaigns.
- Automation of blog post drafts creation with AI-generated images.
- Maintaining content quality and SEO structure automatically.

**Logical Blocks:**

- **1.1 Input Reception:** Fetch pending keywords from Google Sheets and batch process them.
- **1.2 AI Content & Image Generation:** Use an AI agent (LangChain with GPT-4) to generate structured SEO content and invoke a sub-workflow for AI image creation.
- **1.3 Content Quality Control and Expansion:** Check if content meets a 600-word minimum; if not, automatically expand it.
- **1.4 Content Formatting and Merging:** Convert markdown content to HTML, merge with image data, and prepare for Webflow.
- **1.5 Webflow CMS Integration:** Retrieve existing posts, decide to update or create posts based on slug matching, and perform the respective operation.
- **1.6 Results Logging and Error Handling:** Save successful post metadata to Google Sheets and log any failures separately.
- **1.7 AI Image Generation Sub-Workflow:** A separate workflow invoked by the AI agent to generate a single relevant image using Gemini AI and upload it to Google Drive, returning shareable links.

---

### 2. Block-by-Block Analysis

---

#### 1.1 Input Reception

**Overview:**  
Loads all keywords marked as "pending" from a Google Sheets document and processes them in batches for further content generation.

**Nodes Involved:**  
- Schedule Trigger  
- Load Pending Keywords  
- Loop Over Items  
- No Keywords Available  

**Node Details:**  

- **Schedule Trigger**  
  - Type: scheduleTrigger  
  - Role: Initiates the workflow on a monthly interval.  
  - Config: Interval set to months.  
  - Inputs: None  
  - Outputs: Starts the Load Pending Keywords node.  
  - Edge Cases: Trigger failures or misconfiguration of schedule.  

- **Load Pending Keywords**  
  - Type: googleSheets  
  - Role: Reads rows with status "pending" from a Google Sheet.  
  - Config: Filters rows where `status` equals "pending". Uses OAuth2 credentials for access.  
  - Inputs: Trigger from Schedule Trigger  
  - Outputs: List of pending keywords to Loop Over Items.  
  - Edge Cases: Google Sheets API errors, incorrect filter, or credential issues.  

- **Loop Over Items**  
  - Type: splitInBatches  
  - Role: Iterates over each keyword item for sequential processing.  
  - Config: Default batch size (full batch).  
  - Inputs: Pending keywords.  
  - Outputs: Sends each keyword to AI Agent or to No Keywords Available node if empty.  
  - Edge Cases: Empty batch handling, looping errors.  

- **No Keywords Available**  
  - Type: noOp  
  - Role: Ends the workflow gracefully if no keywords are found.  
  - Inputs: From Loop Over Items when no items exist.  
  - Outputs: None.  

---

#### 1.2 AI Content & Image Generation

**Overview:**  
Generates SEO-optimized content and a relevant featured image for each keyword using an AI agent interfacing with GPT-4 and a sub-workflow for image generation.

**Nodes Involved:**  
- Simple Memory  
- OpenAI Chat Model  
- AI Agent  
- AI Image Generation Tool (Sub-workflow Invoke)  
- Process Agent Output  

**Node Details:**  

- **Simple Memory**  
  - Type: langchain.memoryBufferWindow  
  - Role: Maintains conversational context per keyword session.  
  - Config: Session key uses the current keyword.  
  - Inputs: Keyword item from Loop Over Items.  
  - Outputs: Connects to AI Agent node.  
  - Edge Cases: Memory overflow or session mismatch.  

- **OpenAI Chat Model**  
  - Type: langchain.lmChatOpenAi  
  - Role: Provides GPT-4.1-mini model for language generation.  
  - Config: Model set to "gpt-4.1-mini", temperature 0.1 for deterministic output, uses OpenAI credentials.  
  - Inputs: From Simple Memory (language model provider).  
  - Outputs: Sent to AI Agent.  
  - Edge Cases: API rate limits, auth errors, timeout.  

- **AI Agent**  
  - Type: langchain.agent  
  - Role: Coordinates content creation using a detailed prompt specifying SEO structure, keyword usage, and generation of exactly one professional image.  
  - Config: Custom prompt includes instructions for content length, keyword density, section headings, meta description, and image generation call. Output must be valid JSON with strict formatting rules.  
  - Inputs: Language model and memory.  
  - Outputs: Raw JSON content and image generation tool call.  
  - Edge Cases: Invalid JSON output, incomplete generation, exceeding token limits.  

- **AI Image Generation Tool**  
  - Type: langchain.toolWorkflow (Sub-workflow)  
  - Role: Invokes a dedicated sub-workflow to generate a single AI image with Gemini 2.5 Flash model and upload it to Google Drive.  
  - Inputs: Receives imageTitle, imagePrompt, keyword, altText as inputs.  
  - Outputs: Returns imageUrl and alt text for further processing.  
  - Edge Cases: Image generation failure, API errors, upload failures.  
  - Additional Notes: This node calls a separate workflow; see Block 1.7 for details.  

- **Process Agent Output**  
  - Type: code  
  - Role: Cleans and parses the AI Agent’s raw output JSON, calculates word count, and sets status based on word count thresholds.  
  - Config: Includes fallbacks for JSON parsing errors; if parsing fails, returns an error object with safe defaults.  
  - Inputs: Raw AI Agent output.  
  - Outputs: Structured content data with status "content_ready" or "needs_expansion".  
  - Edge Cases: Parsing errors, malformed JSON, missing fields.  

---

#### 1.3 Content Quality Control and Expansion

**Overview:**  
Checks if the generated content meets the 600-word minimum. If not, the content is expanded using GPT-4, preserving structure and keyword optimization.

**Nodes Involved:**  
- Content Quality Check (If)  
- Expand Content (OpenAI node)  
- Format Agent Output (code)  
- Merge Content Paths  

**Node Details:**  

- **Content Quality Check**  
  - Type: if  
  - Role: Checks if `wordCount >= 600` to decide content readiness.  
  - Inputs: Process Agent Output node.  
  - Outputs:  
    - True: proceed to Merge Content Paths.  
    - False: proceed to Expand Content.  
  - Edge Cases: Incorrect word count calculation, expression failures.  

- **Expand Content**  
  - Type: langchain.openAi  
  - Role: Uses GPT-4o-mini model to expand sub-600 word content to 600+ words with detailed instructions to maintain structure and tone.  
  - Config: Temperature 0.3, max tokens 4000, prompts designed to preserve keyword density and formatting. Uses OpenAI credentials.  
  - Inputs: Content needing expansion.  
  - Outputs: Expanded JSON content.  
  - Edge Cases: API limits, invalid JSON output, timeout.  

- **Format Agent Output**  
  - Type: code  
  - Role: Parses expanded content JSON, calculates new word count, and updates status accordingly.  
  - Inputs: Output from Expand Content.  
  - Outputs: Formatted expanded content or error fallback.  
  - Edge Cases: JSON parsing errors.  

- **Merge Content Paths**  
  - Type: merge  
  - Role: Combines outputs from the original path and expanded content path for unified downstream processing.  
  - Inputs: Content Quality Check (true path) and Format Agent Output (expanded path).  
  - Outputs: Proceed to markdown conversion.  

---

#### 1.4 Content Formatting and Merging

**Overview:**  
Transforms markdown content to HTML, merges with image data, and prepares the final payload for Webflow CMS.

**Nodes Involved:**  
- Convert to HTML  
- Get new post slug  
- Get Existing Posts  
- Merge with Existing Posts  

**Node Details:**  

- **Convert to HTML**  
  - Type: markdown  
  - Role: Converts the article content from markdown format to HTML.  
  - Inputs: Merged content JSON.  
  - Outputs: HTML content for Webflow.  
  - Edge Cases: Markdown syntax errors leading to poor HTML conversion.  

- **Get new post slug**  
  - Type: set  
  - Role: Sets the slug field from the article JSON for matching.  
  - Inputs: Convert to HTML output.  
  - Outputs: Used to merge with existing posts.  

- **Get Existing Posts**  
  - Type: webflow  
  - Role: Retrieves all existing posts from the specified Webflow CMS collection.  
  - Config: Uses Webflow OAuth2 credentials with site and collection IDs.  
  - Inputs: None (triggered downstream).  
  - Outputs: Existing posts data.  
  - Edge Cases: API rate limits, authentication failure, collection ID errors.  

- **Merge with Existing Posts**  
  - Type: merge  
  - Role: Merges new content with existing CMS posts by matching slugs to identify whether to update or create.  
  - Inputs: New slug and existing posts.  
  - Outputs: Routed to update or create.  
  - Edge Cases: Slug mismatches, missing keys, merge conflicts.  

---

#### 1.5 Webflow CMS Integration

**Overview:**  
Based on slug existence, updates an existing Webflow post or creates a new one. Then merges the results for success verification.

**Nodes Involved:**  
- Route: Update or Create  
- Update Existing Post  
- Create New Post  
- Merge post result  
- Check Success  

**Node Details:**  

- **Route: Update or Create**  
  - Type: switch  
  - Role: Routes flow based on whether the existing post has an ID (update) or not (create).  
  - Inputs: Merged post data.  
  - Outputs: Two outputs for Update Existing Post or Create New Post.  
  - Edge Cases: Missing or malformed ID fields.  

- **Update Existing Post**  
  - Type: webflow  
  - Role: Updates an existing CMS item with new content, meta description, featured image, and title.  
  - Config: Uses Webflow OAuth2 credentials. Retries up to 3 times on failures.  
  - Inputs: Post ID and content fields.  
  - Outputs: Updated post data.  
  - Edge Cases: API errors, update conflicts, permission issues.  

- **Create New Post**  
  - Type: webflow  
  - Role: Creates a new CMS item with provided content fields.  
  - Config: Similar to update node, with retries and OAuth2 credentials.  
  - Inputs: New post data including slug.  
  - Outputs: Created post data.  
  - Edge Cases: Duplicate slug, permission errors, API limits.  

- **Merge post result**  
  - Type: merge  
  - Role: Combines the results from update or create nodes for unified success/failure handling.  
  - Inputs: Outputs from Update Existing Post and Create New Post.  
  - Outputs: To success check node.  

- **Check Success**  
  - Type: if  
  - Role: Determines if the post creation or update succeeded by checking for a non-empty post ID.  
  - Inputs: Merge post result.  
  - Outputs:  
    - Success: proceed to mark as complete.  
    - Failure: route to error logging.  

---

#### 1.6 Results Logging and Error Handling

**Overview:**  
Logs successful post metadata to a separate Google Sheets document and records errors to a dedicated error log sheet.

**Nodes Involved:**  
- Mark as Complete  
- Save Success Results  
- Log Error  
- Save Error  
- Wait a few seconds  

**Node Details:**  

- **Mark as Complete**  
  - Type: googleSheets  
  - Role: Updates the original keyword row in the keywords sheet, setting status to "created".  
  - Inputs: From Check Success success path.  
  - Outputs: Triggers Save Success Results.  
  - Edge Cases: Google Sheets API errors, row matching failures.  

- **Save Success Results**  
  - Type: googleSheets  
  - Role: Appends or updates content metadata (id, slug, content, meta description, and timestamps) in a "content_created" Google Sheet.  
  - Inputs: From Mark as Complete.  
  - Outputs: Wait a few seconds node.  
  - Edge Cases: API errors, data mismatches.  

- **Wait a few seconds**  
  - Type: wait  
  - Role: Adds a 3-second delay before looping back to process next keyword.  
  - Inputs: From Save Success Results or Save Error.  
  - Outputs: Loops back to Loop Over Items.  

- **Log Error**  
  - Type: code  
  - Role: Prepares error information with keyword, error message, timestamp, and status for logging.  
  - Inputs: From Check Success failure path.  
  - Outputs: Save Error node.  

- **Save Error**  
  - Type: googleSheets  
  - Role: Saves error logs to a dedicated "webflow_error_logs" Google Sheet.  
  - Inputs: From Log Error.  
  - Outputs: Wait a few seconds node.  

---

#### 1.7 AI Image Generation Sub-Workflow

**Overview:**  
A dedicated sub-workflow called by the AI Agent to generate one professional image per article using Gemini 2.5 Flash model, upload it to Google Drive, and return accessible links and metadata.

**Nodes Involved:**  
- When Executed by Another Workflow (Trigger)  
- Generate Image (HTTP Request to OpenRouter API)  
- Process Image Response (Code)  
- Check Image Generation (If)  
- Handle Generation Failure (Code)  
- Convert Base64 to Binary (Code)  
- Upload to Google Drive  
- Get Download Links  
- Result (Set)  

**Node Details:**  

- **When Executed by Another Workflow**  
  - Type: executeWorkflowTrigger  
  - Role: Entry trigger node for sub-workflow, receives image generation parameters from parent workflow.  
  - Inputs: imageTitle, imagePrompt, keyword, altText.  
  - Outputs: To Generate Image node.  

- **Generate Image**  
  - Type: httpRequest  
  - Role: Calls OpenRouter API with Gemini 2.5 Flash model to generate an image from textual prompt.  
  - Config: POST request with JSON body specifying model, prompt, modalities (image + text), and token limits.  
  - Inputs: Parameters from trigger.  
  - Outputs: API response JSON.  
  - Edge Cases: API errors, network issues, invalid response structure.  

- **Process Image Response**  
  - Type: code  
  - Role: Parses API response to extract base64 image URL and related metadata; handles multiple possible response formats.  
  - Outputs: Structured JSON with imageUrl, altText, generation status.  
  - Edge Cases: Parsing errors, missing image URL.  

- **Check Image Generation**  
  - Type: if  
  - Role: Checks if imageUrl is present and non-empty to branch success or failure.  

- **Handle Generation Failure**  
  - Type: code  
  - Role: Prepares failure response JSON when image generation fails, including error details and status.  

- **Convert Base64 to Binary**  
  - Type: code  
  - Role: Converts base64 data URL to binary file data required for Google Drive upload.  
  - Outputs: Binary data with image metadata.  

- **Upload to Google Drive**  
  - Type: googleDrive  
  - Role: Uploads the binary image file to a specified Google Drive folder.  
  - Config: Uses Google Drive OAuth2 credentials, target folder ID set.  
  - Outputs: Google Drive file metadata including webViewLink.  
  - Edge Cases: Upload failures, permission errors.  

- **Get Download Links**  
  - Type: googleDrive  
  - Role: Retrieves permanent download and web view links for the uploaded file.  
  - Inputs: File ID from upload result.  

- **Result**  
  - Type: set  
  - Role: Formats final output including links, alt text, and confirmation messages to return to parent workflow.  

---

### 3. Summary Table

| Node Name                 | Node Type                             | Functional Role                                    | Input Node(s)                         | Output Node(s)                         | Sticky Note                                                                                                                        |
|---------------------------|-------------------------------------|--------------------------------------------------|-------------------------------------|--------------------------------------|-----------------------------------------------------------------------------------------------------------------------------------|
| Schedule Trigger          | scheduleTrigger                     | Starts workflow monthly                          | None                                | Load Pending Keywords                 |                                                                                                                                   |
| Load Pending Keywords      | googleSheets                       | Fetches keywords with status "pending"           | Schedule Trigger                    | Loop Over Items                      |                                                                                                                                   |
| Loop Over Items            | splitInBatches                    | Processes keywords one by one                      | Load Pending Keywords               | No Keywords Available, AI Agent      |                                                                                                                                   |
| No Keywords Available      | noOp                             | Ends workflow if no keywords                       | Loop Over Items                    | None                                |                                                                                                                                   |
| Simple Memory              | langchain.memoryBufferWindow      | Maintains AI context per keyword                   | Loop Over Items                    | AI Agent                           |                                                                                                                                   |
| OpenAI Chat Model          | langchain.lmChatOpenAi           | Provides GPT-4.1-mini language model              | Simple Memory                     | AI Agent                           |                                                                                                                                   |
| AI Agent                  | langchain.agent                  | Generates SEO content + calls image generation    | OpenAI Chat Model, Simple Memory  | Process Agent Output, AI Image Generation Tool | Sticky Note2: AI agent creates full article + generates featured image; one image per article only.                        |
| AI Image Generation Tool   | langchain.toolWorkflow           | Calls sub-workflow for AI image generation        | AI Agent                         | AI Agent                           | Sticky Note18: This is a separate workflow selected as a tool in the AI Agent node. Do not run inside this workflow directly.   |
| Process Agent Output       | code                            | Cleans/parses JSON output, calculates word count  | AI Agent                         | Content Quality Check               |                                                                                                                                   |
| Content Quality Check      | if                              | Checks if content is >= 600 words                  | Process Agent Output               | Merge Content Paths, Expand Content | Sticky Note3: Quality control; expand if under 600 words, else proceed.                                                           |
| Expand Content             | langchain.openAi                | Expands content under 600 words                     | Content Quality Check             | Format Agent Output                |                                                                                                                                   |
| Format Agent Output        | code                            | Parses expanded content JSON, updates status       | Expand Content                   | Merge Content Paths               |                                                                                                                                   |
| Merge Content Paths        | merge                           | Combines original and expanded content paths       | Content Quality Check, Format Agent Output | Convert to HTML                   | Sticky Note4: Converts markdown to HTML and merges with image data.                                                              |
| Convert to HTML            | markdown                        | Converts markdown article content to HTML          | Merge Content Paths              | Get new post slug, Get Existing Posts |                                                                                                                                   |
| Get new post slug          | set                             | Prepares slug for merging with existing posts      | Convert to HTML                 | Merge with Existing Posts         |                                                                                                                                   |
| Get Existing Posts         | webflow                         | Retrieves all existing posts from Webflow CMS      | None (triggered)                | Merge with Existing Posts         | Sticky Note5: Matches slug with Webflow collection to update or create.                                                           |
| Merge with Existing Posts  | merge                           | Joins new content with existing CMS posts by slug | Get new post slug, Get Existing Posts | Route: Update or Create           |                                                                                                                                   |
| Route: Update or Create    | switch                         | Routes to update or create based on post existence | Merge with Existing Posts       | Update Existing Post, Create New Post | Sticky Note6: Updates existing post or creates new one.                                                                           |
| Update Existing Post       | webflow                         | Updates a Webflow CMS item                           | Route: Update or Create         | Merge post result                |                                                                                                                                   |
| Create New Post            | webflow                         | Creates new Webflow CMS item                         | Route: Update or Create         | Merge post result                |                                                                                                                                   |
| Merge post result          | merge                           | Combines update/create results for success check   | Update Existing Post, Create New Post | Check Success                   |                                                                                                                                   |
| Check Success              | if                              | Checks if post creation/update succeeded            | Merge post result              | Mark as Complete, Log Error       | Sticky Note7: Saves success or logs error accordingly.                                                                            |
| Mark as Complete           | googleSheets                    | Marks keyword as "created" in keywords sheet       | Check Success                  | Save Success Results             |                                                                                                                                   |
| Save Success Results       | googleSheets                    | Saves post metadata to "content_created" sheet     | Mark as Complete              | Wait a few seconds               |                                                                                                                                   |
| Log Error                  | code                            | Prepares error log data                             | Check Success                  | Save Error                     |                                                                                                                                   |
| Save Error                 | googleSheets                    | Logs errors to "webflow_error_logs" sheet           | Log Error                    | Wait a few seconds               |                                                                                                                                   |
| Wait a few seconds         | wait                           | Waits 3 seconds between batches                      | Save Success Results, Save Error | Loop Over Items                |                                                                                                                                   |
| When Executed by Another Workflow | executeWorkflowTrigger      | Entry point for AI Image Generation sub-workflow    | Parent workflow               | Generate Image                 | Sticky Note9: AI Image Generation Sub-Workflow details, inputs/outputs, and setup instructions.                                   |
| Generate Image             | httpRequest                    | Calls OpenRouter Gemini 2.5 Flash model API          | When Executed by Another Workflow | Process Image Response        | Sticky Note10: Calls Gemini 2.5 Flash to create image.                                                                             |
| Process Image Response     | code                            | Extracts base64 image URL from API response          | Generate Image               | Check Image Generation          | Sticky Note11: Parses API response for image URL.                                                                                  |
| Check Image Generation     | if                              | Validates presence of image URL                       | Process Image Response        | Convert Base64 to Binary, Handle Generation Failure | Sticky Note12: Routes success or failure based on image URL presence.                                           |
| Handle Generation Failure  | code                            | Handles image generation failure, formats error output | Check Image Generation (fail)  | None                         |                                                                                                                                   |
| Convert Base64 to Binary   | code                            | Converts base64 image data to binary file format      | Check Image Generation (success) | Upload to Google Drive         | Sticky Note13: Converts base64 to binary for upload.                                                                               |
| Upload to Google Drive     | googleDrive                    | Uploads binary image file to Google Drive folder      | Convert Base64 to Binary       | Get Download Links            | Sticky Note14: Uploads image to Google Drive.                                                                                      |
| Get Download Links         | googleDrive                    | Retrieves shareable and download links for uploaded file | Upload to Google Drive        | Result                       | Sticky Note15: Downloads file metadata for links.                                                                                  |
| Result                    | set                             | Formats final output with image URL and alt text     | Get Download Links             | Returns to AI Agent           | Sticky Note16: Final formatting of image output to return to parent workflow.                                                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Trigger Interval: Monthly (1 month)  
   - Purpose: To run the workflow on a schedule.

2. **Add a Google Sheets Node ("Load Pending Keywords")**  
   - Operation: Read rows  
   - Document ID: Your keywords Google Sheet ID  
   - Sheet Name: Specific sheet with keywords  
   - Filter: `status` equals `"pending"`  
   - Credentials: Google Sheets OAuth2   
   - Connect Schedule Trigger → Load Pending Keywords

3. **Add a SplitInBatches Node ("Loop Over Items")**  
   - Purpose: Process each keyword individually  
   - Connect Load Pending Keywords → Loop Over Items

4. **Add a No Operation Node ("No Keywords Available")**  
   - Connect Loop Over Items (empty batch) → No Keywords Available

5. **Add a LangChain Memory Node ("Simple Memory")**  
   - Session Key: `{{$json["main keyword"]}}`  
   - Session ID Type: Custom Key  
   - Connect Loop Over Items (non-empty) → Simple Memory

6. **Add LangChain OpenAI Model Node ("OpenAI Chat Model")**  
   - Model: GPT-4.1-mini  
   - Temperature: 0.1  
   - Credentials: OpenAI API  
   - Connect Simple Memory → OpenAI Chat Model (ai_languageModel)

7. **Add LangChain Agent Node ("AI Agent")**  
   - Prompt: Detailed instructions for SEO content and single image generation with strict JSON output.  
   - System Message: Expert SEO content writer instructions.  
   - Connect OpenAI Chat Model → AI Agent (ai_languageModel)  
   - Connect Simple Memory → AI Agent (ai_memory)  

8. **Add LangChain Tool Workflow Node ("AI Image Generation Tool")**  
   - Select separate workflow for AI image generation (see step 21+)  
   - Inputs: imageTitle, imagePrompt, keyword, altText from current item  
   - Connect AI Agent → AI Image Generation Tool (ai_tool)  

9. **Add Code Node ("Process Agent Output")**  
   - Parse AI Agent output JSON safely  
   - Calculate word count  
   - Set status to "content_ready" or "needs_expansion"  
   - Connect AI Agent → Process Agent Output

10. **Add If Node ("Content Quality Check")**  
    - Condition: wordCount >= 600  
    - True → Merge Content Paths  
    - False → Expand Content  
    - Connect Process Agent Output → Content Quality Check

11. **Add LangChain OpenAI Node ("Expand Content")**  
    - Model: GPT-4o-mini  
    - Temperature: 0.3  
    - Max Tokens: 4000  
    - Prompt: Instructions to expand content to 600+ words preserving structure and style.  
    - Connect Content Quality Check (False) → Expand Content

12. **Add Code Node ("Format Agent Output")**  
    - Parse expanded content JSON  
    - Calculate new word count and update status  
    - Connect Expand Content → Format Agent Output

13. **Add Merge Node ("Merge Content Paths")**  
    - Mode: Combine by position  
    - Inputs: Content Quality Check (True path), Format Agent Output  
    - Connect Content Quality Check (True) → Merge Content Paths (input 1)  
    - Connect Format Agent Output → Merge Content Paths (input 2)

14. **Add Markdown Node ("Convert to HTML")**  
    - Mode: Markdown to HTML  
    - Input: content field from merged data  
    - Connect Merge Content Paths → Convert to HTML

15. **Add Set Node ("Get new post slug")**  
    - Set field "slug" to `{{$json.slug}}`  
    - Connect Convert to HTML → Get new post slug

16. **Add Webflow Node ("Get Existing Posts")**  
    - Operation: Get All  
    - Site ID and Collection ID: Set to your Webflow site and CMS collection  
    - Credentials: Webflow OAuth2  
    - Connect Convert to HTML → Get Existing Posts

17. **Add Merge Node ("Merge with Existing Posts")**  
    - Mode: Keep Everything  
    - Merge by fields: slug from new post and slug from existing posts  
    - Connect Get new post slug → Merge with Existing Posts (input 1)  
    - Connect Get Existing Posts → Merge with Existing Posts (input 2)

18. **Add Switch Node ("Route: Update or Create")**  
    - Condition: if id field exists and not empty → Update Existing Post  
    - Else → Create New Post  
    - Connect Merge with Existing Posts → Route: Update or Create

19. **Add Webflow Node ("Update Existing Post")**  
    - Operation: Update  
    - Item ID: `{{$json.id}}`  
    - Fields: name, page-content, metadescription, featured-image from converted HTML data  
    - Credentials: Webflow OAuth2  
    - Connect Route: Update or Create (Update path) → Update Existing Post

20. **Add Webflow Node ("Create New Post")**  
    - Operation: Create  
    - Fields: name, slug, page-content, metadescription, featured-image  
    - Credentials: Webflow OAuth2  
    - Connect Route: Update or Create (Create path) → Create New Post

21. **Add Merge Node ("Merge post result")**  
    - Combines update and create results  
    - Connect Update Existing Post and Create New Post → Merge post result

22. **Add If Node ("Check Success")**  
    - Condition: id field is not empty  
    - True → Mark as Complete  
    - False → Log Error  
    - Connect Merge post result → Check Success

23. **Add Google Sheets Node ("Mark as Complete")**  
    - Operation: Append or Update  
    - Matches keyword in keywords sheet, sets status to "created"  
    - Credentials: Google Sheets OAuth2  
    - Connect Check Success (True) → Mark as Complete

24. **Add Google Sheets Node ("Save Success Results")**  
    - Operation: Append or Update  
    - Saves post metadata to content_created sheet (id, slug, content, meta description, timestamps)  
    - Credentials: Google Sheets OAuth2  
    - Connect Mark as Complete → Save Success Results

25. **Add Code Node ("Log Error")**  
    - Prepares error log object with keyword, error, timestamp  
    - Connect Check Success (False) → Log Error

26. **Add Google Sheets Node ("Save Error")**  
    - Operation: Append or Update  
    - Saves error logs to webflow_error_logs sheet  
    - Credentials: Google Sheets OAuth2  
    - Connect Log Error → Save Error

27. **Add Wait Node ("Wait a few seconds")**  
    - Wait time: 3 seconds  
    - Connect Save Success Results → Wait a few seconds  
    - Connect Save Error → Wait a few seconds  
    - Connect Wait a few seconds → Loop Over Items (to continue processing next keyword)

---

### AI Image Generation Sub-Workflow Setup

28. **Create a new workflow with an Execute Workflow Trigger Node ("When Executed by Another Workflow")**  
    - Inputs: imageTitle, imagePrompt, keyword, altText

29. **Add HTTP Request Node ("Generate Image")**  
    - POST to `https://openrouter.ai/api/v1/chat/completions`  
    - JSON Body includes: model "google/gemini-2.5-flash-image-preview", prompt with imagePrompt, modalities image+text  

30. **Add Code Node ("Process Image Response")**  
    - Extract base64 image URL from API response  
    - Return imageUrl, keyword, altText, status

31. **Add If Node ("Check Image Generation")**  
    - Check if imageUrl exists and is non-empty  
    - True → Convert Base64 to Binary  
    - False → Handle Generation Failure

32. **Add Code Node ("Handle Generation Failure")**  
    - Format failure response JSON with error message

33. **Add Code Node ("Convert Base64 to Binary")**  
    - Convert base64 string to binary data for upload  
    - Prepare binary data with appropriate filename and MIME type

34. **Add Google Drive Node ("Upload to Google Drive")**  
    - Uploads image binary to specified Google Drive folder  
    - Credentials: Google Drive OAuth2

35. **Add Google Drive Node ("Get Download Links")**  
    - Retrieve webView and download links for uploaded file  

36. **Add Set Node ("Result")**  
    - Formats output with image URLs, alt text, and messages for return to parent workflow  

37. **Connect nodes sequentially:**  
    When Executed by Another Workflow → Generate Image → Process Image Response → Check Image Generation → (success) Convert Base64 to Binary → Upload to Google Drive → Get Download Links → Result  
    → (failure) Handle Generation Failure → Result

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                      | Context or Link                                                                                                                        |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------|
| This workflow generates SEO-optimized articles at scale and saves them as Webflow drafts. It reads keywords from Google Sheets, writes 600+ word articles with proper structure, generates custom images via Gemini AI, and tracks results. | Sticky Note at workflow start with overview                                                                                          |
| Before running this workflow, configure Webflow OAuth2 credentials in n8n as per instructions: create app in Webflow, enable Data Client REST API, copy OAuth Redirect URL from n8n, and set site/collection IDs.                    | Sticky Note near Webflow nodes                                                                                                       |
| The AI Image Generation is handled by a separate sub-workflow which must be created individually and selected in the AI Agent node as a tool. Do not run the image generation workflow inside the main workflow.                   | Sticky Note near AI Image Generation Tool node (Sticky Note18)                                                                         |
| The OpenRouter API key and Google Drive folder ID must be configured in the AI Image Generation sub-workflow before use.                                                                                                        | Sticky Note9 near sub-workflow entry node                                                                                            |
| Recommended to test with a single keyword before running bulk processing.                                                                                                                                                        | Sticky Note at workflow start                                                                                                        |
| The AI Agent strictly requires JSON output without markdown or extra text to correctly parse content; failures fall back to error reporting.                                                                                   | AI Agent prompt and Process Agent Output node behavior                                                                              |

---

**Disclaimer:** The text and data described are generated and processed exclusively via an automated workflow created with n8n, adhering to all relevant content policies and legal guidelines. All data manipulated is lawful and publicly accessible.

---