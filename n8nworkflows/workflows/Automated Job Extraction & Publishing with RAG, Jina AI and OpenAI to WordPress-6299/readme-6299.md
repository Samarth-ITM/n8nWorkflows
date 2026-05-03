Automated Job Extraction & Publishing with RAG, Jina AI and OpenAI to WordPress

https://n8nworkflows.xyz/workflows/automated-job-extraction---publishing-with-rag--jina-ai-and-openai-to-wordpress-6299


# Automated Job Extraction & Publishing with RAG, Jina AI and OpenAI to WordPress

### 1. Workflow Overview

This workflow automates the extraction, processing, and publishing of job listings from URLs received via Telegram messages. It uses a Retrieval-Augmented Generation (RAG) approach combining Jina AI for web data extraction, OpenAI for natural language processing, and Supabase as a vector database for knowledge retrieval. The processed job data is then formatted and published as posts on a WordPress site.

**Target Use Cases:**  
- Automating the intake of job posting URLs shared via Telegram.  
- Extracting job details from web pages using AI tools.  
- Regenerating job posts following strict formatting and style rules stored in a knowledge base.  
- Publishing well-formatted job listings to WordPress with company logos.  
- Handling errors and retries with fallback mechanisms to ensure data completeness.

**Logical Blocks:**  
- **1.1 Input Reception and URL Preparation:** Receive Telegram messages with job URLs, validate and prepare URLs for extraction.  
- **1.2 Web Data Extraction & Knowledge Retrieval:** Extract job listing content from URLs using Jina AI and augment with RAG via Supabase vector store.  
- **1.3 AI Processing & Data Extraction:** Use OpenAI GPT-4.1-mini with strict formatting instructions to extract structured job data.  
- **1.4 Data Mapping & Validation:** Map extracted job categories and types to predefined IDs; check completeness of required fields.  
- **1.5 Logo Handling:** Download company logos and upload them to WordPress media library.  
- **1.6 Final Data Formatting:** Format post data for WordPress according to extracted and mapped information.  
- **1.7 Publishing & Notifications:** Publish job listing posts to WordPress and send Telegram notifications about success or failure.  
- **1.8 Error Handling & Retry:** Manage errors, fallback to alternative logic if data incomplete, and notify users accordingly.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and URL Preparation

**Overview:**  
Receives new job URLs via Telegram, validates URL format, and prepares the URL string for extraction.

**Nodes Involved:**  
- 📥 New Job Link via Telegram  
- 🔧 Prepare URL for Extraction  
- 📚 Load Valid Job Types & Categories  
- valid url?  
- notify: wrong url  
- notify: processing job  
- Wait  

**Node Details:**  

- **📥 New Job Link via Telegram**  
  - Type: Telegram Trigger  
  - Role: Entry point to receive Telegram messages containing job URLs.  
  - Config: Listens for "message" updates.  
  - Outputs: Raw Telegram message JSON including text.  
  - Potential Failures: Webhook misconfigurations, Telegram API errors.  

- **🔧 Prepare URL for Extraction**  
  - Type: Set node  
  - Role: Extracts and cleans the URL from Telegram message text; creates a "cleanUrl" variable containing the base URL or "Invalid URL".  
  - Expressions: Extracts URL prefix from message text using string manipulation and regex trim.  
  - Outputs: JSON with cleaned URL and raw text.  
  - Edge Cases: Malformed or missing URLs; fallback to "Invalid URL".  

- **📚 Load Valid Job Types & Categories**  
  - Type: Set node  
  - Role: Loads static arrays of valid job categories and types for downstream validation and mapping.  
  - Outputs: Two arrays: `category` and `types` with id-name mappings.  
  - Edge Cases: None (static data).  

- **valid url?**  
  - Type: If node  
  - Role: Checks if the cleaned URL is valid (not "Invalid URL").  
  - Conditions: String not equals "Invalid URL".  
  - Outputs: Branches to either continue processing or notify wrong URL.  

- **notify: wrong url**  
  - Type: Telegram node  
  - Role: Sends Telegram message to user if URL is invalid.  
  - Config: Sends HTML formatted message prompting user to check input.  
  - Edge Cases: Telegram API failures.  

- **notify: processing job**  
  - Type: Telegram node  
  - Role: Sends acknowledgement message to user that job processing has started.  
  - Additional: Includes user first name for personalization.  

- **Wait**  
  - Type: Wait node  
  - Role: Adds a delay (1.25 seconds) before proceeding, possibly to stagger processing or avoid API rate limits.  

---

#### 1.2 Web Data Extraction & Knowledge Retrieval

**Overview:**  
Extract job listing content from the provided URL using Jina AI extraction, with fallback monitoring and vector store updates.

**Nodes Involved:**  
- notify: extracting  
- Extract (Jina AI)  
- If (valid extraction)  
- notify: success extract  
- notify: failed extract  
- Loop Over Items  
- Supabase Vector Store  
- Embeddings OpenAI  
- Default Data Loader  
- Recursive Character Text Splitter  

**Node Details:**  

- **notify: extracting**  
  - Telegram node notifying user that web data extraction is in progress.  

- **Extract**  
  - Type: Jina AI node  
  - Role: Extracts structured data from the URL content, including title, description, metadata, and icons.  
  - Inputs: URL from "Prepare URL" node.  
  - Outputs: Extracted JSON data including external icon URLs.  
  - Edge Cases: Extraction failure, API timeouts; configured to continue on error.  

- **If (valid extraction)**  
  - Checks if essential fields (title, description, URL, content) exist in extraction result.  
  - Branch: Success continues workflow; failure triggers notification.  

- **notify: success extract**  
  - Notifies user that the job extraction succeeded and processing with OpenAI starts.  

- **notify: failed extract**  
  - Notifies user that extraction failed and advises to inform maintainer.  

- **Loop Over Items**  
  - Splits multi-item extraction results into batches to process individually.  

- **Supabase Vector Store**  
  - Inserts documents into Supabase vector database for retrieval-augmented generation.  
  - Used for knowledge retrieval by AI nodes.  

- **Embeddings OpenAI**  
  - Generates embedding vectors for input texts via OpenAI embeddings API.  

- **Default Data Loader**  
  - Loads binary data for document processing downstream.  

- **Recursive Character Text Splitter**  
  - Splits large text documents into smaller chunks for better embedding and retrieval.  

---

#### 1.3 AI Processing & Data Extraction

**Overview:**  
Uses OpenAI GPT with strict RAG-based instructions to extract and regenerate job data in a structured JSON format.

**Nodes Involved:**  
- Supabase Vector Store1  
- Embeddings OpenAI1  
- 🧠 Extract Job Data with GPT  
- 🧮 Map Job Type & Category IDs  
- notify: openai success  

**Node Details:**  

- **Supabase Vector Store1**  
  - Retrieves relevant documents from Supabase to serve as knowledge base for AI prompt.  
  - Acts as a tool integrated into AI prompt.  

- **Embeddings OpenAI1**  
  - Creates embeddings for the query to perform vector similarity search.  

- **🧠 Extract Job Data with GPT**  
  - Type: OpenAI node (GPT-4.1-mini)  
  - Role: Processes extracted data and retrieved knowledge base content to produce structured job fields like title, job_type, categories, location, company, logo, original_link, and regenerated HTML content.  
  - Prompt: Includes detailed instructions to strictly follow formatting and style rules from the knowledge base ("document" tool), including HTML structure, SEO, and language guidelines.  
  - Output: JSON with extracted fields, including HTML-formatted job description.  
  - Failures: OpenAI API errors, prompt misinterpretation; configured to continue on error.  

- **🧮 Map Job Type & Category IDs**  
  - Type: Code node  
  - Role: Maps extracted category and job_type strings to predefined numeric IDs required by WordPress taxonomy.  
  - Logic: Uses static arrays to match case-sensitive categories and types, filtering invalid entries.  
  - Output: JSON object with arrays of IDs for categories and types.  

- **notify: openai success**  
  - Telegram notification confirming OpenAI processing completed successfully.  

---

#### 1.4 Data Mapping & Validation

**Overview:**  
Validates presence of all required job post fields and branches workflow accordingly; handles missing data with fallback attempts.

**Nodes Involved:**  
- 📦 Format Final Job Post Data  
- ✅ All Fields Available?  
- if not valid  
- Edit Fields  
- ✅ All Fields Available? - Alt  

**Node Details:**  

- **📦 Format Final Job Post Data**  
  - Type: Set node  
  - Role: Collects fields extracted by AI and mapped IDs into final JSON structure expected by WordPress REST API.  
  - Fields include title, content (HTML), job_listing_type, job_listing_category, location, company name, company logo URL, application URL, and status ("publish").  

- **✅ All Fields Available?**  
  - If node  
  - Role: Checks that all critical fields (title, category, type, location, application, companyLogo) are not empty.  
  - Branches: Valid data leads to publishing; invalid triggers fallback logic.  

- **if not valid**  
  - Triggers fallback mechanism when validation fails. Sends message indicating retry attempt.  

- **Edit Fields**  
  - Adjusts or supplements missing fields with fallback values to improve data completeness.  

- **✅ All Fields Available? - Alt**  
  - Secondary validation after applying fallback field edits.  

---

#### 1.5 Logo Handling

**Overview:**  
Downloads company logo images from extracted URLs and uploads them to WordPress media library to associate with job posts.

**Nodes Involved:**  
- 📥 Download Company Logo  
- ☁️ Upload Logo to WordPress  
- 📥 Download Company Logo - Alt  
- ☁️ Upload Logo to WordPress - Alt  

**Node Details:**  

- **📥 Download Company Logo**  
  - HTTP Request node  
  - Role: Downloads logo image file from URL extracted by AI.  
  - Configured to continue on error (fallback).  

- **☁️ Upload Logo to WordPress**  
  - HTTP Request node  
  - Role: Uploads downloaded logo to WordPress media endpoint using predefined OAuth2 credentials.  
  - Sets HTTP headers for Content-Disposition and Content-Type for correct file handling.  

- **📥 Download Company Logo - Alt**  
  - Alternative download node used in fallback path with edited URLs.  

- **☁️ Upload Logo to WordPress - Alt**  
  - Alternative upload node used in fallback path.  

- **Edge Cases:**  
  - Image URL unavailable or invalid; fallback to default logo URL in final post data.  
  - Upload failures retried or notified as errors.  

---

#### 1.6 Final Data Formatting

**Overview:**  
Prepares the final job post payload for WordPress REST API including all fields and metadata.

**Nodes Involved:**  
- 📦 Format Final Job Post Data  
- 📦 Format Final Job Post Data - Alt  

**Node Details:**  

- Both nodes aggregate extracted, mapped, and logo information into structured JSON for WordPress job listing endpoint.  
- Alternative node used in fallback scenario with adjusted logo data.  
- Uses expressions to pull data from AI extraction and mapping nodes.  

---

#### 1.7 Publishing & Notifications

**Overview:**  
Publishes the formatted job listing to WordPress and sends Telegram notifications about success or failure.

**Nodes Involved:**  
- 🚀 Publish to Your web  
- 🚀 Publish to Yourweb - Alt  
- 📊 Did Publish Succeed?1  
- 📊 Did Publish Succeed?2  
- notify: success extract  
- notify: failed extract  
- notify: openai success  
- notify: processing job  
- notify: error  
- notify: wrong url  
- notify: failed extract  
- notify: openai success  
- notify: openai success  
- notify: openai success  
- Publiched!  
- 📨 Notify Success - Alt  
- 📨 Notify Failed - Alt  
- Error  
- changing method  

**Node Details:**  

- **🚀 Publish to Your web** and **🚀 Publish to Yourweb - Alt**  
  - HTTP Request nodes posting job data to WordPress REST API with OAuth credentials.  
  - JSON body includes title, content, status, job types, categories, and metadata including logo URL.  
  - Retry on failure enabled.  

- **📊 Did Publish Succeed?1** and **📊 Did Publish Succeed?2**  
  - If nodes checking presence of critical response fields (id, link, date) and categories to confirm successful post creation.  
  - Branches to success or failure notifications.  

- **notify: success extract**, **notify: processing job**, **notify: openai success**, **notify: error**, **notify: wrong url**, **notify: failed extract**  
  - Telegram nodes sending appropriate status messages at different stages.  

- **Publiched!**, **📨 Notify Success - Alt**, **📨 Notify Failed - Alt**, **Error**, **changing method**  
  - Various Telegram notification nodes for user feedback on workflow state and fallback attempts.  

- Edge Cases:  
  - WordPress API failures, network issues, missing response data leading to fallback or error notifications.  

---

#### 1.8 Error Handling & Retry

**Overview:**  
Captures workflow errors, sends formatted error reports via Telegram, and manages fallback retry logic for incomplete data.

**Nodes Involved:**  
- Error Trigger  
- Code (Error Formatter)  
- Kirim ke Telegram (Send Error Report)  
- Wait1, Wait2  
- changing method  
- if not valid  

**Node Details:**  

- **Error Trigger**  
  - Captures execution errors globally.  

- **Code**  
  - Formats error details including workflow name, node, message, timestamp, execution link into an HTML Telegram message.  

- **Kirim ke Telegram**  
  - Sends formatted error message to a monitoring Telegram chat.  

- **Wait1**, **Wait2**  
  - Delay nodes used before retry attempts.  

- **changing method**  
  - Sends Telegram message indicating fallback attempt underway with default values.  

- **if not valid**  
  - Branches to fallback logic when validation fails.  

- Edge Cases:  
  - Unhandled exceptions, API rate limits, malformed data inputs.  

---

### 3. Summary Table

| Node Name                       | Node Type                        | Functional Role                         | Input Node(s)                         | Output Node(s)                       | Sticky Note                                                                                       |
|--------------------------------|---------------------------------|---------------------------------------|-------------------------------------|------------------------------------|-------------------------------------------------------------------------------------------------|
| 📥 New Job Link via Telegram    | Telegram Trigger                | Entry point for job URLs via Telegram | -                                   | 🔧 Prepare URL for Extraction       |                                                                                                 |
| 🔧 Prepare URL for Extraction   | Set                            | Extracts and cleans URL from Telegram | 📥 New Job Link via Telegram          | 📚 Load Valid Job Types & Categories |                                                                                                 |
| 📚 Load Valid Job Types & Categories | Set                        | Loads static valid job categories/types | 🔧 Prepare URL for Extraction         | valid url?                         |                                                                                                 |
| valid url?                     | If                             | Checks URL validity                    | 📚 Load Valid Job Types & Categories  | notify: processing job / notify: wrong url |                                                                                                 |
| notify: wrong url              | Telegram                       | Notifies invalid URL error            | valid url? (false branch)             | -                                  |                                                                                                 |
| notify: processing job         | Telegram                       | Notifies job processing started       | valid url? (true branch)              | Wait                              |                                                                                                 |
| Wait                          | Wait                          | Delay before extraction                | notify: processing job                | notify: extracting                 |                                                                                                 |
| notify: extracting             | Telegram                       | Notifies extraction in progress       | Wait                               | Extract                          |                                                                                                 |
| Extract                       | Jina AI                       | Extracts job data from URL             | notify: extracting                   | If (valid extraction)              |                                                                                                 |
| If (valid extraction)          | If                             | Checks extraction success              | Extract                            | notify: success extract / notify: failed extract |                                                                                                 |
| notify: success extract       | Telegram                       | Notifies successful extraction         | If (valid extraction)                | 🧠 Extract Job Data with GPT       |                                                                                                 |
| notify: failed extract        | Telegram                       | Notifies extraction failure            | If (valid extraction) (false branch) | -                                  |                                                                                                 |
| 🧠 Extract Job Data with GPT    | OpenAI                        | Extracts structured job data with RAG | notify: success extract              | 🧮 Map Job Type & Category IDs      |                                                                                                 |
| 🧮 Map Job Type & Category IDs  | Code                         | Maps categories and job types to IDs  | 🧠 Extract Job Data with GPT          | notify: openai success             |                                                                                                 |
| notify: openai success        | Telegram                       | Notifies OpenAI processing success    | 🧮 Map Job Type & Category IDs        | Create Url                       |                                                                                                 |
| Create Url                    | Code                         | Extracts icon URL and processes data  | notify: openai success               | 📥 Download Company Logo            |                                                                                                 |
| 📥 Download Company Logo       | HTTP Request                 | Downloads company logo image           | Create Url                        | ☁️ Upload Logo to WordPress         |                                                                                                 |
| ☁️ Upload Logo to WordPress     | HTTP Request                 | Uploads logo to WordPress media        | 📥 Download Company Logo             | 📦 Format Final Job Post Data       |                                                                                                 |
| 📦 Format Final Job Post Data   | Set                          | Formats final job post JSON for WP     | ☁️ Upload Logo to WordPress           | ✅ All Fields Available?           |                                                                                                 |
| ✅ All Fields Available?        | If                           | Validates presence of required fields  | 📦 Format Final Job Post Data         | 🚀 Publish to Your web / if not valid / Error |                                                                                                 |
| if not valid                  | Telegram / Wait / Set         | Handles fallback on missing data       | ✅ All Fields Available? (false branch) | Wait2 / Edit Fields / Error        |                                                                                                 |
| Wait2                         | Wait                         | Delay before retry                      | if not valid                       | changing method                   |                                                                                                 |
| changing method               | Telegram                      | Notifies retry with fallback values    | Wait2                            | Wait1                            |                                                                                                 |
| Wait1                        | Wait                         | Delay before retry                      | changing method                   | notify: extracting (retry loop)    |                                                                                                 |
| Edit Fields                  | Set                          | Edits fields to fill missing data       | if not valid                      | 📥 Download Company Logo - Alt      |                                                                                                 |
| 📥 Download Company Logo - Alt | HTTP Request                 | Alternative logo download for fallback | Edit Fields                     | ☁️ Upload Logo to WordPress - Alt   |                                                                                                 |
| ☁️ Upload Logo to WordPress - Alt | HTTP Request              | Alternative logo upload for fallback   | 📥 Download Company Logo - Alt      | 📦 Format Final Job Post Data - Alt |                                                                                                 |
| 📦 Format Final Job Post Data - Alt | Set                     | Formats post data for fallback path    | ☁️ Upload Logo to WordPress - Alt    | ✅ All Fields Available? - Alt      |                                                                                                 |
| ✅ All Fields Available? - Alt  | If                           | Validates fallback post data            | 📦 Format Final Job Post Data - Alt  | 🚀 Publish to Yourweb - Alt / 📨 Notify Failed - Alt |                                                                                                 |
| 🚀 Publish to Your web         | HTTP Request                 | Publishes job listing to WordPress      | ✅ All Fields Available?            | 📊 Did Publish Succeed?2           |                                                                                                 |
| 🚀 Publish to Yourweb - Alt    | HTTP Request                 | Publishes job listing fallback to WordPress | ✅ All Fields Available? - Alt     | 📊 Did Publish Succeed?1           |                                                                                                 |
| 📊 Did Publish Succeed?2       | If                           | Checks success of primary publish       | 🚀 Publish to Your web             | Publiched! / Error                |                                                                                                 |
| 📊 Did Publish Succeed?1       | If                           | Checks success of fallback publish      | 🚀 Publish to Yourweb - Alt        | 📨 Notify Success - Alt / 📨 Notify Failed - Alt |                                                                                                 |
| Publiched!                   | Telegram                      | Notifies successful primary publish     | 📊 Did Publish Succeed?2 (true)    | -                                |                                                                                                 |
| 📨 Notify Success - Alt        | Telegram                      | Notifies successful fallback publish    | 📊 Did Publish Succeed?1 (true)    | -                                |                                                                                                 |
| 📨 Notify Failed - Alt         | Telegram                      | Notifies failed fallback publish        | 📊 Did Publish Succeed?1 (false)   | -                                |                                                                                                 |
| Error                        | Telegram                      | Notifies failure in publishing           | 📊 Did Publish Succeed?2 (false)   | -                                |                                                                                                 |
| Error Trigger                | Error Trigger                | Captures workflow errors                 | -                                | Code                            |                                                                                                 |
| Code (Error Formatter)        | Code                         | Formats error message for Telegram      | Error Trigger                    | Kirim ke Telegram                |                                                                                                 |
| Kirim ke Telegram            | Telegram                      | Sends formatted error report             | Code                            | -                                |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger Node:**  
   - Name: "📥 New Job Link via Telegram"  
   - Type: Telegram Trigger  
   - Listen for "message" updates.  
   - Use Telegram API credentials with bot token.  

2. **Create Set Node to Prepare URL:**  
   - Name: "🔧 Prepare URL for Extraction"  
   - Extract `text` or `message.text` from Telegram data.  
   - Generate `cleanUrl` by extracting protocol + domain from the text or mark as "Invalid URL".  

3. **Create Set Node for Valid Job Types & Categories:**  
   - Name: "📚 Load Valid Job Types & Categories"  
   - Assign two static arrays:  
     - `category`: Array of category objects with `id` and `category` string.  
     - `types`: Array of job type objects with `id` and `type` string.  

4. **Create If Node to Validate URL:**  
   - Name: "valid url?"  
   - Condition: Check `cleanUrl` != "Invalid URL".  

5. **Create Telegram Node to Notify Wrong URL:**  
   - Name: "notify: wrong url"  
   - Send message to Telegram user indicating invalid URL.  

6. **Create Telegram Node to Notify Processing Started:**  
   - Name: "notify: processing job"  
   - Send message to user confirming job processing start.  

7. **Create Wait Node:**  
   - Name: "Wait"  
   - Delay ~1.25 seconds before extraction.  

8. **Create Telegram Node to Notify Extraction:**  
   - Name: "notify: extracting"  
   - Inform user extraction is in progress.  

9. **Create Jina AI Node to Extract Data:**  
   - Name: "Extract"  
   - Input: `cleanUrl` from previous nodes.  
   - Credentials: Jina AI API key.  
   - Output: Extracted job listing data including metadata and icons.  

10. **Create If Node to Check Extraction Success:**  
    - Name: "If"  
    - Check presence of `title`, `description`, `url`, and `content` fields.  

11. **Create Telegram Nodes:**  
    - "notify: success extract" (on extraction success)  
    - "notify: failed extract" (on extraction failure)  

12. **Create Supabase Vector Store Node (retrieve mode):**  
    - Name: "Supabase Vector Store1"  
    - Retrieve relevant knowledge base documents for RAG.  
    - Credentials: Supabase API with appropriate table.  

13. **Create OpenAI Embeddings Node:**  
    - Name: "Embeddings OpenAI1"  
    - Input: Query text, credentials for OpenAI.  

14. **Create OpenAI Chat Node:**  
    - Name: "🧠 Extract Job Data with GPT"  
    - Model: GPT-4.1-mini (or latest)  
    - Prompt includes detailed instructions for extraction and formatting based on RAG dataset and retrieved knowledge base.  
    - Output: JSON with structured job fields and regenerated HTML content.  

15. **Create Code Node to Map Job Type & Category IDs:**  
    - Name: "🧮 Map Job Type & Category IDs"  
    - Map extracted string fields to numeric IDs using predefined arrays from Step 3.  
    - Output arrays `fixCategory` and `fixType`.  

16. **Create Telegram Node to Notify OpenAI Success:**  
    - Name: "notify: openai success"  

17. **Create Code Node to Extract Icon URL:**  
    - Name: "Create Url"  
    - Extract icon URL from Jina AI extraction result with error handling.  

18. **Create HTTP Request Node to Download Logo:**  
    - Name: "📥 Download Company Logo"  
    - Download image from extracted icon URL.  
    - Set to continue on error.  

19. **Create HTTP Request Node to Upload Logo to WordPress:**  
    - Name: "☁️ Upload Logo to WordPress"  
    - POST to WordPress media endpoint with OAuth2 credentials.  
    - Set headers for binary upload.  

20. **Create Set Node to Format Final Job Post Data:**  
    - Name: "📦 Format Final Job Post Data"  
    - Combine extracted fields and mapped IDs into JSON for WordPress post creation.  
    - Include fallback default logo URL if missing.  

21. **Create If Node to Validate All Required Fields:**  
    - Name: "✅ All Fields Available?"  
    - Check presence and non-emptiness of title, categories, type, location, application URL, and logo GUID.  

22. **Create Branches:**  
    - If valid, proceed to publishing.  
    - If invalid, send Telegram message, wait, and attempt fallback:  
      - "if not valid" -> "Wait2" -> "changing method" -> "Wait1" -> "notify: extracting" (retry cycle).  

23. **Create Set Node to Edit Fields for Fallback:**  
    - Name: "Edit Fields"  
    - Concatenate or replace missing logo URL or other fields for fallback.  

24. **Create Alternative Download and Upload Logo Nodes:**  
    - "📥 Download Company Logo - Alt"  
    - "☁️ Upload Logo to WordPress - Alt"  

25. **Create Alternative Format Final Post Data Node:**  
    - Name: "📦 Format Final Job Post Data - Alt"  

26. **Create Alternative Validation Node:**  
    - Name: "✅ All Fields Available? - Alt"  

27. **Create HTTP Request Nodes to Publish Posts:**  
    - Name: "🚀 Publish to Your web" (primary)  
    - Name: "🚀 Publish to Yourweb - Alt" (fallback)  
    - POST job post JSON to WordPress jobs endpoint.  

28. **Create If Nodes to Confirm Successful Publish:**  
    - "📊 Did Publish Succeed?2" for primary  
    - "📊 Did Publish Succeed?1" for fallback  

29. **Create Telegram Notification Nodes for Publish Results:**  
    - On success: "Publiched!", "📨 Notify Success - Alt"  
    - On failure: "Error", "📨 Notify Failed - Alt"  

30. **Create Error Handling Nodes:**  
    - "Error Trigger" to catch workflow errors globally.  
    - "Code" node to format error messages with workflow and node details.  
    - "Kirim ke Telegram" node sends error reports to monitoring chat.  

31. **Create Additional Telegram Notifications** throughout for user feedback on progress and errors.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------|
| The workflow uses RAG (Retrieval-Augmented Generation) combining Supabase vector store and OpenAI GPT to strictly follow formatting/style rules in job posts. | Core concept driving AI extraction and regeneration quality.                  |
| Telegram messages serve as the main user input and notification channel, ensuring interactive status updates.                                                 | Telegram API and bot credentials required.                                   |
| WordPress REST API is used for media uploads and job listing publishing with OAuth2 authentication.                                                           | WordPress site must support REST API endpoints and OAuth2 credentials.       |
| Fallback mechanisms include retrying with default logo URL and edited fields to improve robustness and data completeness.                                      | Prevents workflow failure on partial data or extraction errors.              |
| Sticky Notes in the original workflow provide usage tips and guide node editing; see n8n documentation for sticky note usage.                                  | https://docs.n8n.io/workflows/sticky-notes/                                  |
| Contact for support or errors is specified as "khairul" with WhatsApp link in notifications.                                                                   | https://wa.me/6285155431253                                                  |
| The workflow is currently inactive; enable and test in a controlled environment before production use.                                                        | Workflow status: inactive                                                    |

---

**Disclaimer:**  
This documentation is based exclusively on an n8n automated workflow. It adheres strictly to content policies and contains no illegal or protected data. All handled data is legal and public.