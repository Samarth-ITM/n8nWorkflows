Automatic Job Listings Extraction and Publishing Template

https://n8nworkflows.xyz/workflows/automatic-job-listings-extraction-and-publishing-template-6518


# Automatic Job Listings Extraction and Publishing Template

### 1. Workflow Overview

This workflow automates the extraction, processing, and publishing of job listings from URLs received via Telegram messages. It targets use cases where job offers are shared as links and need to be parsed, structured, and published efficiently on a WordPress website, specifically tailored for digital art, animation, and related fields. The workflow leverages Google Drive for file storage, Supabase for vector storage, OpenAI for advanced data extraction and content regeneration, and Telegram for user interaction and notifications.

The workflow is organized into the following logical blocks:

- **1.1 Input Reception and URL Preparation:** Receives new job links via Telegram, prepares and validates URLs for processing.
- **1.2 Content Extraction and Web Scraping:** Extracts raw job offer content using an external scraping API.
- **1.3 AI Data Extraction and Processing:** Uses OpenAI GPT and vector database tools to extract structured job data and regenerate the job description in HTML format.
- **1.4 Data Mapping and Validation:** Maps extracted job categories and types to predefined IDs, checks for required fields.
- **1.5 Company Logo Handling:** Downloads company logos and uploads them to WordPress.
- **1.6 Job Post Formatting and Publishing:** Formats the final job post data and publishes it to a WordPress job listings site.
- **1.7 Notifications and Monitoring:** Sends Telegram notifications on each key step, handles errors, and retries failed attempts.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and URL Preparation

- **Overview:**  
  This block listens for new job listing URLs via Telegram, extracts the URL from the message, validates it, and prepares it for further processing.

- **Nodes Involved:**  
  - 📥 New Job Link via Telegram  
  - 🔧 Prepare URL for Extraction  
  - 📚 Load Valid Job Types & Categories  
  - valid url?  
  - notify: wrong url  
  - notify: processing job  

- **Node Details:**

  - **📥 New Job Link via Telegram**  
    - Type: Telegram Trigger  
    - Role: Receives Telegram updates (messages), triggering workflow on new job links.  
    - Config: Listens to "message" updates, uses Telegram API credentials.  
    - Outputs: Message text containing job URL.  
    - Failure: Possible webhook issues or API throttling.

  - **🔧 Prepare URL for Extraction**  
    - Type: Set  
    - Role: Extracts and cleans the URL from the Telegram message text.  
    - Config: Sets two fields: `text` (the original message text) and `cleanUrl` (protocol + domain extracted from the URL or "Invalid URL" if malformed).  
    - Expressions: Uses JavaScript expressions to split and clean URL.  
    - Inputs: Telegram message JSON.  
    - Outputs: Clean URL string for validation.  
    - Failure: Expression errors if message text malformed.

  - **📚 Load Valid Job Types & Categories**  
    - Type: Set  
    - Role: Loads predefined arrays of valid job categories and job types used for mapping and validation later.  
    - Config: Hardcoded arrays of categories (with IDs) and job types (with IDs).  
    - Outputs: Provides reference arrays for validation.  
    - Failure: N/A.

  - **valid url?**  
    - Type: If  
    - Role: Checks if the cleaned URL is valid (not "Invalid URL").  
    - Config: Condition to check if `cleanUrl` ≠ "Invalid URL".  
    - Outputs: Branches to either continue processing or notify invalid URL.  
    - Failure: N/A.

  - **notify: wrong url**  
    - Type: Telegram  
    - Role: Sends a Telegram message informing that the URL is invalid.  
    - Config: Sends message with HTML formatting and no attribution.  
    - Failure: Telegram API errors.

  - **notify: processing job**  
    - Type: Telegram  
    - Role: Notifies the user that job processing has started.  
    - Config: Sends a personalized message using the user's first name.  
    - Failure: Telegram API errors.

#### 1.2 Content Extraction and Web Scraping

- **Overview:**  
  This stage extracts the job listing content from the provided URL by calling an external scraping API, then verifies the extraction success.

- **Nodes Involved:**  
  - notify: extracting  
  - Extract  
  - If (check extraction success)  
  - notify: success extract  
  - notify: failed extract  
  - notify: error  

- **Node Details:**

  - **notify: extracting**  
    - Type: Telegram  
    - Role: Updates user that web data extraction is in progress.  
    - Config: Edits previous Telegram message with italic HTML text.  
    - Failure: Telegram errors.

  - **Extract**  
    - Type: HTTP Request  
    - Role: Calls the Firecrawl scraping API with the job URL to get markdown content.  
    - Config: POST request with Authorization header (API key required), sending URL and requesting markdown format.  
    - Failure: API errors, authentication failure, network timeouts.

  - **If (check extraction success)**  
    - Type: If  
    - Role: Checks if the extraction response contains required fields (`title`, `description`, `url`, `content`).  
    - Outputs: Success path or failure path.  
    - Failure: Logic errors if response structure changes.

  - **notify: success extract**  
    - Type: Telegram  
    - Role: Notifies the user that extraction was successful and AI processing will start.  
    - Config: Edits message with confirmation.  
    - Failure: Telegram errors.

  - **notify: failed extract**  
    - Type: Telegram  
    - Role: Notifies the user that extraction failed and developer should be informed.  
    - Failure: Telegram errors.

  - **notify: error**  
    - Type: Telegram  
    - Role: Generic error notification.  
    - Failure: Telegram errors.

#### 1.3 AI Data Extraction and Processing

- **Overview:**  
  Uses OpenAI GPT to extract structured job data (title, categories, location, company, logo URL, etc.) and regenerates the job description HTML according to strict formatting rules stored in a vector knowledge base.

- **Nodes Involved:**  
  - Supabase Vector Store1 (retrieve-as-tool)  
  - Embeddings OpenAI1  
  - 🧠 Extract Job Data with GPT  
  - 🧮 Map Job Type & Category IDs  
  - notify: openai success  
  - Create Url  

- **Node Details:**

  - **Supabase Vector Store1**  
    - Type: LangChain Vector Store (Supabase) - retrieve as tool  
    - Role: Searches the vector knowledge base ("documents" table) to retrieve formatting and style guides for AI prompt context.  
    - Config: Uses "match_documents" query to find relevant docs.  
    - Failure: Supabase connection/auth errors.

  - **Embeddings OpenAI1**  
    - Type: LangChain OpenAI Embeddings  
    - Role: Generates embeddings for query to use in Supabase vector search.  
    - Failure: OpenAI API errors, quota limits.

  - **🧠 Extract Job Data with GPT**  
    - Type: LangChain OpenAI Node (GPT-4.1-mini)  
    - Role: Extracts structured info from scraped job content using retrieved document context. Contains detailed system prompt with extraction and formatting instructions in Spanish.  
    - Config: Uses messages with system and user roles, JSON output expected.  
    - Failure: OpenAI API errors, prompt formatting errors, unexpected responses.

  - **🧮 Map Job Type & Category IDs**  
    - Type: Code  
    - Role: Maps extracted `categories` and `job_type` strings to numeric IDs using predefined arrays.  
    - Config: JavaScript mapping logic with filtering and fallback.  
    - Outputs: JSON with `fixCategory` and `fixType` arrays of IDs.  
    - Failure: Mapping errors if inputs missing or mismatched.

  - **notify: openai success**  
    - Type: Telegram  
    - Role: Notifies user that OpenAI processing succeeded and next steps are underway.  
    - Failure: Telegram errors.

  - **Create Url**  
    - Type: Code  
    - Role: Extracts the company logo URL from the external icon object safely, handling errors and fallback.  
    - Failure: JS errors if input structure changes.

#### 1.4 Data Mapping and Validation

- **Overview:**  
  Validates presence of all required extracted fields, formats final post data, and branches workflow for error handling or retry with default logo.

- **Nodes Involved:**  
  - 📦 Format Final Job Post Data  
  - ✅ All Fields Available?  
  - if not valid  
  - Edit Fields  
  - Error  

- **Node Details:**

  - **📦 Format Final Job Post Data**  
    - Type: Set  
    - Role: Builds the final data structure (title, content, job types, categories, location, company info, application URL, company logo) for WordPress publishing.  
    - Values: Uses extracted GPT data and mapped IDs; sets default logo fallback URL.  
    - Failure: Missing fields will be caught downstream.

  - **✅ All Fields Available?**  
    - Type: If  
    - Role: Checks that critical fields (`title`, `job_listing_category`, `job_listing_type`, `_job_location`, `_application`, `companyLogo`) are all present and non-empty.  
    - Outputs: Success path to publishing or failure path to retry with defaults.  
    - Failure: Logic errors if false negatives occur.

  - **if not valid**  
    - Type: Telegram  
    - Role: Sends Telegram message to user indicating missing fields and triggers secondary retry approach.  
    - Failure: Telegram errors.

  - **Edit Fields**  
    - Type: Set  
    - Role: Combines trimmed URL and logo fallback for the retry alternate path.  
    - Failure: Expression errors if source data missing.

  - **Error**  
    - Type: Telegram  
    - Role: Sends failure notification if all attempts fail.  
    - Failure: Telegram errors.

#### 1.5 Company Logo Handling

- **Overview:**  
  Downloads company logos from extracted URLs and uploads them to WordPress media library to be used in job posts.

- **Nodes Involved:**  
  - 📥 Download Company Logo  
  - ☁️ Upload Logo to WordPress  
  - 📥 Download Company Logo - Alt  
  - ☁️ Upload Logo to WordPress - Alt  

- **Node Details:**

  - **📥 Download Company Logo**  
    - Type: HTTP Request  
    - Role: Downloads logo image binary from processed icon URL.  
    - Config: Response format set to file.  
    - OnError: Continues workflow despite errors to allow fallback.  
    - Failure: URL invalid, network issues.

  - **☁️ Upload Logo to WordPress**  
    - Type: HTTP Request  
    - Role: Uploads downloaded logo to WordPress media endpoint.  
    - Config: POST with binary data, sets Content-Disposition and Content-Type headers dynamically with company name, uses WordPress API credentials.  
    - Retry enabled.  
    - Failure: WordPress auth errors, upload errors.

  - **📥 Download Company Logo - Alt**  
    - Same as above but uses fallback combined URL from Edit Fields node.

  - **☁️ Upload Logo to WordPress - Alt**  
    - Same as above, alternative upload path.

#### 1.6 Job Post Formatting and Publishing

- **Overview:**  
  Finalizes job post data, publishes to WordPress job listings endpoint, checks publishing success, and sends user notifications accordingly.

- **Nodes Involved:**  
  - 📦 Format Final Job Post Data - Alt  
  - ✅ All Fields Available? - Alt  
  - 🚀 Publish to Your web  
  - 🚀 Publish to Yourweb - Alt  
  - 📊 Did Publish Succeed?2  
  - 📊 Did Publish Succeed?1  
  - Publiched!  
  - 📨 Notify Success - Alt  
  - 📨 Notify Failed - Alt  

- **Node Details:**

  - **📦 Format Final Job Post Data - Alt**  
    - Type: Set  
    - Role: Formats final job post data for alternate path with default/fallback logo and fields.  
    - Failure: Same as primary.

  - **✅ All Fields Available? - Alt**  
    - Type: If  
    - Role: Validates required fields on alternate data.  
    - Outputs: Proceed to publish or notify failure.  
    - Failure: Logic errors.

  - **🚀 Publish to Your web**  
    - Type: HTTP Request  
    - Role: Publishes job post to WordPress via REST API (job-listings endpoint).  
    - Config: POST, JSON body constructed dynamically from final data, uses WordPress credentials.  
    - Retry enabled.  
    - Failure: Auth errors, API errors, invalid data errors.

  - **🚀 Publish to Yourweb - Alt**  
    - Same as above but for alternate data path.

  - **📊 Did Publish Succeed?2**  
    - Type: If  
    - Role: Checks if primary publish returned valid IDs and link.  
    - Outputs: Success notification or error node.  
    - Failure: Logic errors.

  - **📊 Did Publish Succeed?1**  
    - Type: If  
    - Role: Checks if alternate publish succeeded.  
    - Outputs: Success or failure notifications.  
    - Failure: Logic errors.

  - **Publiched!**  
    - Type: Telegram  
    - Role: Sends success notification to user with job post details.  
    - Failure: Telegram API errors.

  - **📨 Notify Success - Alt**  
    - Type: Telegram  
    - Role: Alternate path success notification.  
    - Failure: Telegram errors.

  - **📨 Notify Failed - Alt**  
    - Type: Telegram  
    - Role: Failure notification with contact info.  
    - Failure: Telegram errors.

#### 1.7 Notifications and Monitoring

- **Overview:**  
  Provides error handling, logging, and monitoring throughout the workflow using Telegram messages and error triggers.

- **Nodes Involved:**  
  - Error Trigger  
  - Code (error message formatting)  
  - Kirim ke Telegram (send error message)  
  - Sticky Note (RAG Data)  
  - Sticky Note2 (Monitoring)  

- **Node Details:**

  - **Error Trigger**  
    - Type: Error Trigger  
    - Role: Catches any errors occurring in the workflow.  
    - Failure: N/A.

  - **Code**  
    - Type: Code  
    - Role: Formats detailed error message with workflow, node, error description, timestamp, execution link, and level.  
    - Failure: Errors in JS code unlikely but possible.

  - **Kirim ke Telegram**  
    - Type: Telegram  
    - Role: Sends formatted error message to a predefined Telegram chat for monitoring.  
    - Failure: Telegram errors.

  - **Sticky Notes**  
    - Provide documentation and monitoring hints for workflow users.  
    - Content includes a guide link for RAG data and a note to leave monitoring nodes as-is.

---

### 3. Summary Table

| Node Name                         | Node Type                         | Functional Role                                      | Input Node(s)                        | Output Node(s)                                   | Sticky Note                                                                                  |
|----------------------------------|----------------------------------|-----------------------------------------------------|------------------------------------|-------------------------------------------------|----------------------------------------------------------------------------------------------|
| Google Drive Trigger (disabled)  | Google Drive Trigger              | Trigger on file updates in Google Drive              | -                                  | Search File                                      |                                                                                              |
| Search File                      | Google Drive                     | Lists files in Google Drive folder                    | Google Drive Trigger               | Get Data                                        |                                                                                              |
| Get Data                        | Google Drive                     | Downloads file content                                | Search File                       | Loop Over Items                                 |                                                                                              |
| Loop Over Items                  | Split In Batches                 | Processes each file item batch                         | Get Data                         | Supabase Vector Store (on success)              |                                                                                              |
| Supabase Vector Store            | LangChain Vector Store Supabase | Inserts document embeddings into Supabase vector DB  | Loop Over Items                  | Loop Over Items                                 |                                                                                              |
| Recursive Character Text Splitter| LangChain Text Splitter          | Splits text into smaller chunks for embedding         | Default Data Loader              | Default Data Loader                              |                                                                                              |
| Default Data Loader              | LangChain Document Loader        | Loads documents from binary data                       | Recursive Character Text Splitter | Supabase Vector Store                            |                                                                                              |
| Embeddings OpenAI               | LangChain Embeddings OpenAI      | Generates embeddings for documents                     | Default Data Loader              | Supabase Vector Store                            |                                                                                              |
| 📥 New Job Link via Telegram      | Telegram Trigger                 | Receives new job URL messages                          | -                                | 🔧 Prepare URL for Extraction                    |                                                                                              |
| 🔧 Prepare URL for Extraction      | Set                             | Extracts and cleans URL from Telegram message          | 📥 New Job Link via Telegram      | 📚 Load Valid Job Types & Categories             |                                                                                              |
| 📚 Load Valid Job Types & Categories| Set                             | Loads predefined valid job types and categories        | 🔧 Prepare URL for Extraction      | valid url?                                       |                                                                                              |
| valid url?                      | If                              | Checks if URL is valid                                 | 📚 Load Valid Job Types & Categories| notify: processing job / notify: wrong url     |                                                                                              |
| notify: wrong url               | Telegram                        | Notifies user about invalid URL                         | valid url?                      | -                                               |                                                                                              |
| notify: processing job          | Telegram                        | Notifies user that job processing started               | valid url?                      | Wait                                           |                                                                                              |
| Wait                           | Wait                           | Waits 1.25 seconds                                     | notify: processing job           | notify: extracting                              |                                                                                              |
| notify: extracting              | Telegram                        | Notifies user that extraction is ongoing                | Wait                           | Extract                                         |                                                                                              |
| Extract                        | HTTP Request                   | Calls external scraping API to extract job content      | notify: extracting              | If                                              |                                                                                              |
| If                            | If                              | Validates extraction success                             | Extract                        | notify: success extract / notify: failed extract|                                                                                              |
| notify: success extract        | Telegram                        | Notifies user of successful extraction                   | If (success)                   | 🧠 Extract Job Data with GPT                      |                                                                                              |
| notify: failed extract         | Telegram                        | Notifies user of extraction failure                      | If (failure)                   | -                                               |                                                                                              |
| 🧠 Extract Job Data with GPT      | LangChain OpenAI               | Extracts structured job data and regenerates HTML       | notify: success extract          | 🧮 Map Job Type & Category IDs / notify: error   |                                                                                              |
| Supabase Vector Store1          | LangChain Vector Store Supabase | Retrieves formatting/style guides for GPT prompt         | Embeddings OpenAI1              | 🧠 Extract Job Data with GPT                      |                                                                                              |
| Embeddings OpenAI1             | LangChain Embeddings OpenAI      | Generates embeddings for retrieval query                 | Supabase Vector Store1          | Supabase Vector Store1                           |                                                                                              |
| 🧮 Map Job Type & Category IDs     | Code                           | Maps extracted categories and job types to numeric IDs  | 🧠 Extract Job Data with GPT      | notify: openai success                           |                                                                                              |
| notify: openai success         | Telegram                        | Notifies user that OpenAI processing succeeded           | 🧮 Map Job Type & Category IDs   | Create Url                                      |                                                                                              |
| Create Url                    | Code                           | Extracts and processes company logo URL                   | notify: openai success          | 📥 Download Company Logo                         |                                                                                              |
| 📥 Download Company Logo          | HTTP Request                   | Downloads company logo from URL                            | Create Url                     | ☁️ Upload Logo to WordPress                      |                                                                                              |
| ☁️ Upload Logo to WordPress       | HTTP Request                   | Uploads company logo image to WordPress media library     | 📥 Download Company Logo          | 📦 Format Final Job Post Data                    |                                                                                              |
| 📦 Format Final Job Post Data      | Set                             | Formats job post data for publishing                       | ☁️ Upload Logo to WordPress       | ✅ All Fields Available?                         |                                                                                              |
| ✅ All Fields Available?          | If                              | Checks required post fields presence                       | 📦 Format Final Job Post Data     | 🚀 Publish to Your web / if not valid / Error   |                                                                                              |
| if not valid                  | Telegram                        | Notifies user about missing fields and triggers retry     | ✅ All Fields Available?          | Wait2                                           |                                                                                              |
| Wait2                         | Wait                           | Waits 2 seconds before retrying alternate path             | if not valid                   | changing method                                 |                                                                                              |
| changing method               | Telegram                        | Notifies user about retry using default values             | Wait2                         | Wait1                                           |                                                                                              |
| Wait1                         | Wait                           | Waits 2 seconds before alternate logo download             | changing method                | 📥 Download Company Logo - Alt                   |                                                                                              |
| Edit Fields                  | Set                             | Combines trimmed URL and fallback logo URL for retry       | if not valid                   | 📥 Download Company Logo - Alt                   |                                                                                              |
| 📥 Download Company Logo - Alt     | HTTP Request                   | Downloads alternate logo for retry                          | Edit Fields                   | ☁️ Upload Logo to WordPress - Alt                |                                                                                              |
| ☁️ Upload Logo to WordPress - Alt  | HTTP Request                   | Uploads alternate logo to WordPress media library           | 📥 Download Company Logo - Alt    | 📦 Format Final Job Post Data - Alt              |                                                                                              |
| 📦 Format Final Job Post Data - Alt | Set                             | Formats job post data with alternate logo or defaults       | ☁️ Upload Logo to WordPress - Alt | ✅ All Fields Available? - Alt                  |                                                                                              |
| ✅ All Fields Available? - Alt    | If                              | Checks required fields on alternate data                    | 📦 Format Final Job Post Data - Alt | 🚀 Publish to Yourweb - Alt / 📨 Notify Failed - Alt |                                                                                              |
| 🚀 Publish to Your web           | HTTP Request                   | Publishes job post to WordPress endpoint (primary)          | ✅ All Fields Available?          | 📊 Did Publish Succeed?2                         |                                                                                              |
| 🚀 Publish to Yourweb - Alt       | HTTP Request                   | Publishes job post using alternate data                     | ✅ All Fields Available? - Alt    | 📊 Did Publish Succeed?1                         |                                                                                              |
| 📊 Did Publish Succeed?2          | If                              | Validates success of primary publishing                      | 🚀 Publish to Your web           | Publiched! / Error                               |                                                                                              |
| 📊 Did Publish Succeed?1          | If                              | Validates success of alternate publishing                    | 🚀 Publish to Yourweb - Alt      | 📨 Notify Success - Alt / 📨 Notify Failed - Alt |                                                                                              |
| Publiched!                    | Telegram                        | Sends success notification to user                            | 📊 Did Publish Succeed?2          | -                                               |                                                                                              |
| 📨 Notify Success - Alt          | Telegram                        | Sends success notification for alternate path                | 📊 Did Publish Succeed?1          | -                                               |                                                                                              |
| 📨 Notify Failed - Alt           | Telegram                        | Sends failure notification with contact info                  | ✅ All Fields Available? - Alt    | -                                               |                                                                                              |
| Error Trigger                 | Error Trigger                  | Catches workflow errors                                      | -                                | Code                                            |                                                                                              |
| Code                         | Code                           | Formats detailed error message                                | Error Trigger                  | Kirim ke Telegram                               |                                                                                              |
| Kirim ke Telegram             | Telegram                        | Sends error message to monitoring Telegram chat               | Code                          | -                                               |                                                                                              |
| Sticky Note                  | Sticky Note                    | Documentation note with guide link                            | -                                | -                                               | ## RAG DATA - **Double click** to edit me. [Guide](https://docs.n8n.io/workflows/sticky-notes/) |
| Sticky Note2                 | Sticky Note                    | Monitoring note advising not to modify                        | -                                | -                                               | ## Monitoring - leave as it is                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Telegram Trigger:**  
   - Node Name: 📥 New Job Link via Telegram  
   - Type: Telegram Trigger  
   - Config: Listen for "message" updates, use Telegram API credentials.

2. **Prepare URL:**  
   - Node Name: 🔧 Prepare URL for Extraction  
   - Type: Set  
   - Assign `text` = message text or fallback, `cleanUrl` = extract protocol + domain or "Invalid URL" if not valid using JavaScript expressions.

3. **Load Valid Job Types & Categories:**  
   - Node Name: 📚 Load Valid Job Types & Categories  
   - Type: Set  
   - Set two arrays: categories (with IDs), types (with IDs) as per predefined list.

4. **Validate URL:**  
   - Node Name: valid url?  
   - Type: If  
   - Condition: `cleanUrl` not equal to "Invalid URL".

5. **Notify Wrong URL:**  
   - Node Name: notify: wrong url  
   - Type: Telegram  
   - Send message indicating invalid URL.

6. **Notify Processing Start:**  
   - Node Name: notify: processing job  
   - Type: Telegram  
   - Send message greeting user and indicating job processing.

7. **Wait Node:**  
   - Node Name: Wait  
   - Type: Wait  
   - Duration: 1.25 seconds.

8. **Notify Extracting:**  
   - Node Name: notify: extracting  
   - Type: Telegram  
   - Edit previous message to show "Extracting web data..." with HTML italic.

9. **Extract Content:**  
   - Node Name: Extract  
   - Type: HTTP Request  
   - POST to Firecrawl API with URL and Authentication header (Bearer token)  
   - Request JSON body: `{ "url": "{{cleanUrl}}", "formats": ["markdown"] }`.

10. **Check Extraction:**  
    - Node Name: If  
    - Type: If  
    - Condition: Check fields `title`, `description`, `url`, and `content` exist in response.

11. **Notify Extraction Success/Failure:**  
    - Node Name: notify: success extract / notify: failed extract  
    - Type: Telegram  
    - Send appropriate messages.

12. **Retrieve Knowledge Base:**  
    - Node Name: Supabase Vector Store1  
    - Type: LangChain Vector Store (Supabase)  
    - Mode: retrieve-as-tool  
    - Table: documents  
    - Credentials: Supabase API.

13. **Generate Embeddings:**  
    - Node Name: Embeddings OpenAI1  
    - Type: LangChain Embeddings OpenAI  
    - Credentials: OpenAI API.

14. **Extract Job Data with GPT:**  
    - Node Name: 🧠 Extract Job Data with GPT  
    - Type: LangChain OpenAI  
    - Model: GPT-4.1-mini  
    - Messages: System and user prompt as per detailed instructions including strict formatting rules, category/type restrictions, and content extraction.  
    - Credentials: OpenAI API.

15. **Map Categories and Types:**  
    - Node Name: 🧮 Map Job Type & Category IDs  
    - Type: Code  
    - JS mapping logic from extracted strings to numeric IDs arrays.

16. **Notify OpenAI Success:**  
    - Node Name: notify: openai success  
    - Type: Telegram

17. **Create Processed Logo URL:**  
    - Node Name: Create Url  
    - Type: Code  
    - Extract icon URL safely from external icon object.

18. **Download Company Logo:**  
    - Node Name: 📥 Download Company Logo  
    - Type: HTTP Request  
    - Download binary file from processed icon URL (response format: file).

19. **Upload Logo to WordPress:**  
    - Node Name: ☁️ Upload Logo to WordPress  
    - Type: HTTP Request  
    - POST to WordPress media API, send binary data, set headers for filename and content-type, use WordPress credentials.

20. **Format Final Job Post Data:**  
    - Node Name: 📦 Format Final Job Post Data  
    - Type: Set  
    - Compose final job post fields: title, content (HTML), job types, categories, location, company, logo URL, application link, status="publish".

21. **Validate All Fields:**  
    - Node Name: ✅ All Fields Available?  
    - Type: If  
    - Ensure required fields are non-empty.

22. **Branch on Validation:**  
    - Success: Proceed to publish.  
    - Failure: Send Telegram notify and attempt alternate path (Edit Fields, alternate download/upload logo, format alternate post).

23. **Publish to WordPress:**  
    - Node Name: 🚀 Publish to Your web  
    - Type: HTTP Request  
    - POST to WordPress job listings endpoint with JSON body from formatted data, WordPress credentials.

24. **Check Publish Success:**  
    - Node Name: 📊 Did Publish Succeed?2  
    - Type: If  
    - Validate response contains post ID, link, categories, and date.

25. **Send Success/Failure Notifications:**  
    - Nodes: Publiched!, Error, notify: error, notify: failed extract, notify: failed alt, notify: success alt, etc.

26. **Error Handling and Monitoring:**  
    - Error Trigger node to catch errors globally.  
    - Code node to format error messages.  
    - Telegram node to send errors to monitoring chat.

27. **Sticky Notes:**  
    - Add documentation notes and monitoring instructions as sticky notes on canvas.

---

### 5. General Notes & Resources

| Note Content                                                                                                                | Context or Link                                                  |
|-----------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------|
| ## RAG DATA - **Double click** to edit me. [Guide](https://docs.n8n.io/workflows/sticky-notes/)                              | Sticky Note on RAG Data storage and usage                       |
| ## Monitoring - leave as it is                                                                                              | Sticky Note on monitoring nodes                                 |
| Workflow uses Firecrawl API for web scraping: requires a valid API key (replace placeholder in Extract node).               | Firecrawl API documentation: https://firecrawl.dev/            |
| WordPress REST API endpoint URL placeholders (e.g. https://www.yourweb.es/wp-json/wp/v2/job-listings) must be replaced.     | WordPress REST API docs: https://developer.wordpress.org/rest-api/ |
| Telegram API credentials require bot token and chat IDs configured properly.                                                | Telegram Bot API docs: https://core.telegram.org/bots/api       |
| Supabase credentials must be configured with appropriate permissions for vector store and knowledge base usage.             | Supabase docs: https://supabase.com/docs                        |
| OpenAI API credentials require access to GPT-4.1-mini or equivalent model.                                                  | OpenAI API docs: https://platform.openai.com/docs/models       |
| The workflow includes fallback logic for missing logos and retries using default images to avoid publishing failures.      |                                                                |
| All notifications are sent in Spanish with personalized user data, reflecting the target audience and workflow purpose.    |                                                                |
| The workflow is designed for automated job listings extraction for digital arts and animation-related job boards.          |                                                                |

---

**Disclaimer:** The provided workflow is an automated n8n workflow designed for legal and public data processing strictly following content policies. It contains no illegal or offensive material and manipulates only authorized data.