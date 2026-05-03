Generate & Schedule SEO Blog Posts for Multiple Clients with Gemini AI & Elementor

https://n8nworkflows.xyz/workflows/generate---schedule-seo-blog-posts-for-multiple-clients-with-gemini-ai---elementor-7168


# Generate & Schedule SEO Blog Posts for Multiple Clients with Gemini AI & Elementor

### 1. Workflow Overview

This workflow automates the generation and scheduling of SEO-optimized blog posts for multiple clients by leveraging Google Gemini AI and Elementor for WordPress integration. It is designed to handle client data input, generate blog content using AI, format the content for Elementor, create associated images, authenticate with WordPress, and finally post and schedule blogs in WordPress while updating Google Sheets records to track progress.

The workflow is logically divided into the following blocks:

- **1.1 Input Data Acquisition:** Triggers and retrieves client and content data from Google Sheets or form submissions.
- **1.2 Data Preparation and Mapping:** Processes and formats client data and prompts for AI consumption.
- **1.3 AI Content Generation:** Uses Google Gemini Chat Model and AI Agent nodes to generate blog content.
- **1.4 Content Formatting and Image Generation:** Converts AI-generated content to Elementor-compatible HTML and generates related images.
- **1.5 WordPress Authentication and Posting:** Authenticates with WordPress, uploads images, and posts the blog content.
- **1.6 Scheduling and Tracking:** Manages blog scheduling logic and updates Google Sheets with status and metadata.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Data Acquisition

- **Overview:**  
  This block handles the triggering of the workflow via schedule or form submission and fetches client and blog-related data from Google Sheets.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Google Sheets Trigger  
  - Get all values (Google Sheets)  
  - Get row(s) in sheet (Google Sheets)  
  - Loop Over Items (Split in Batches)  
  - On form submission1 (Form Trigger)

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow on a defined schedule (e.g., daily) for batch processing of multiple clients.  
    - Inputs: None  
    - Outputs: Triggers "Get row(s) in sheet" to fetch client data rows.  
    - Failures: Misconfiguration of schedule or time zone issues.

  - **Google Sheets Trigger**  
    - Type: Google Sheets Trigger  
    - Role: Triggers the workflow when a change or new entry occurs in the Google Sheet (event-driven start).  
    - Inputs: None  
    - Outputs: Starts "Get all values" to fetch current sheet data.  
    - Failures: Authentication errors or sheet access issues.

  - **Get all values**  
    - Type: Google Sheets  
    - Role: Retrieves all values from a configured Google Sheet (likely client or blog data).  
    - Inputs: Trigger from Google Sheets Trigger  
    - Outputs: Feeds "Format to 'values'".  
    - Failures: API rate limits, sheet permissions.

  - **Get row(s) in sheet**  
    - Type: Google Sheets  
    - Role: Retrieves specific rows based on schedule criteria for processing.  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Feeds "Loop Over Items".  
    - Failures: Range or sheet errors.

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes retrieved rows in batches to avoid overload and handle each client/blog data sequentially.  
    - Inputs: Rows from "Get row(s) in sheet"  
    - Outputs: To "Map Data" for data preparation; has an unused output path as well.  
    - Failures: Batch sizing issues or empty input data.

  - **On form submission1**  
    - Type: Form Trigger  
    - Role: An alternative trigger for receiving blog post requests via form submission.  
    - Inputs: Webhook trigger on form submit  
    - Outputs: Directly feeds "Map Data".  
    - Failures: Webhook misconfiguration or network issues.

---

#### 1.2 Data Preparation and Mapping

- **Overview:**  
  This block formats and maps the input data (from sheets or form) into a structure suitable for AI prompt generation.

- **Nodes Involved:**  
  - Map Data (Set)  
  - Modify Prompt (Code)  
  - Client (Google Sheets)  
  - Replace values (Set)  
  - Format to 'values' (Set)  
  - Write JSON (Code)

- **Node Details:**

  - **Map Data**  
    - Type: Set  
    - Role: Maps input fields (e.g., client info, blog topic) into variables for prompt construction.  
    - Inputs: From "Loop Over Items" or "On form submission1"  
    - Outputs: Feeds "Modify Prompt".  
    - Key Expressions: Likely uses expressions to map specific columns or form fields.

  - **Modify Prompt**  
    - Type: Code (JavaScript)  
    - Role: Customizes the AI prompt by dynamically inserting client data and blog specifics.  
    - Inputs: Mapped data from "Map Data"  
    - Outputs: Feeds "Client" node (to fetch client details).  
    - Key Expressions: JavaScript code manipulating prompt strings.

  - **Client**  
    - Type: Google Sheets  
    - Role: Fetches detailed client information if needed, such as SEO keywords or preferences.  
    - Inputs: From "Modify Prompt"  
    - Outputs: Feeds "AI Agent".  
    - Failures: Sheet access or missing data.

  - **Replace values**  
    - Type: Set  
    - Role: Updates or replaces placeholder values in output data, likely for tracking or finalizing data before sheet update.  
    - Inputs: From "n8n | get wf" (workflow data retrieval)  
    - Outputs: Feeds "n8n | update".  

  - **Format to 'values'**  
    - Type: Set  
    - Role: Formats data into the 'values' structure required by Google Sheets API for updating rows.  
    - Inputs: From "Get all values"  
    - Outputs: Feeds "Write JSON".

  - **Write JSON**  
    - Type: Code  
    - Role: Prepares or logs JSON-formatted data for debugging or further processing.  
    - Inputs: From "Format to 'values'"  
    - Outputs: Feeds "n8n | get wf".

---

#### 1.3 AI Content Generation

- **Overview:**  
  This block uses Google Gemini’s chat model combined with an AI Agent node to generate SEO blog content based on prepared prompts.

- **Nodes Involved:**  
  - Google Gemini Chat Model  
  - AI Agent  
  - Simple Memory

- **Node Details:**

  - **Google Gemini Chat Model**  
    - Type: LangChain LM Chat Google Gemini  
    - Role: Calls Google Gemini AI to generate conversational or textual content from the prompt.  
    - Inputs: Configured prompt from previous nodes.  
    - Outputs: AI-generated text fed into "AI Agent".  
    - Failures: API quota, network issues.

  - **AI Agent**  
    - Type: LangChain Agent  
    - Role: Manages multi-step AI interactions or chains, possibly handling complex prompt logic or memory context.  
    - Inputs: From "Google Gemini Chat Model" and memory buffer  
    - Outputs: Feeds "HTML to Elementor Format".  
    - Retry on fail enabled to handle transient errors.  
    - Failures: Expression errors or AI service downtime.

  - **Simple Memory**  
    - Type: LangChain Memory Buffer Window  
    - Role: Maintains conversational context or prompt history for the AI Agent.  
    - Inputs: AI Agent output linked as memory.  
    - Outputs: Feeds back to AI Agent.  
    - Failures: Memory overflow or data corruption.

---

#### 1.4 Content Formatting and Image Generation

- **Overview:**  
  This block transforms AI-generated blog content into Elementor-compatible HTML and triggers image generation for blog posts.

- **Nodes Involved:**  
  - HTML to Elementor Format (Code)  
  - WordpressAuth (Code)  
  - Generate an image (Google Gemini - LangChain)  
  - Upload Image (HTTP Request)  
  - WordpressData (Set)

- **Node Details:**

  - **HTML to Elementor Format**  
    - Type: Code (JavaScript)  
    - Role: Converts plain HTML or AI content into format compatible with Elementor page builder.  
    - Inputs: From "AI Agent"  
    - Outputs: Feeds "WordpressAuth".  
    - Failures: Code errors or invalid HTML input.

  - **WordpressAuth**  
    - Type: Code  
    - Role: Handles OAuth2 or other authentication flows to authorize posting on WordPress sites.  
    - Inputs: From "HTML to Elementor Format"  
    - Outputs: Feeds "Generate an image".  
    - Failures: Auth token expiration, invalid credentials.

  - **Generate an image**  
    - Type: LangChain Google Gemini  
    - Role: Calls AI to generate images related to the blog post topic or content.  
    - Inputs: From "WordpressAuth"  
    - Outputs: Feeds "Upload Image".  
    - Failures: API limits or generation failures.

  - **Upload Image**  
    - Type: HTTP Request  
    - Role: Uploads generated image to WordPress media library.  
    - Inputs: From "Generate an image"  
    - Outputs: Feeds "WordpressData".  
    - Failures: Network errors, invalid upload URL.

  - **WordpressData**  
    - Type: Set  
    - Role: Prepares the final data payload including formatted content and image URLs for blog posting.  
    - Inputs: From "Upload Image"  
    - Outputs: Feeds "Post Blog".  

---

#### 1.5 WordPress Posting

- **Overview:**  
  This block posts the prepared blog content to WordPress and appends relevant metadata to Google Sheets.

- **Nodes Involved:**  
  - Post Blog (HTTP Request)  
  - Append row in sheet (Google Sheets)  
  - If Scheduled (If)  
  - Update row in sheet (Google Sheets)  
  - n8n | get wf (n8n workflow node)  
  - n8n | update (n8n workflow node)

- **Node Details:**

  - **Post Blog**  
    - Type: HTTP Request  
    - Role: Sends a POST request to WordPress REST API to create a new blog post with formatted content and images.  
    - Inputs: From "WordpressData"  
    - Outputs: Feeds "Append row in sheet".  
    - Failures: API errors, invalid payload, or network issues.

  - **Append row in sheet**  
    - Type: Google Sheets  
    - Role: Adds a new row to the Google Sheet to log the published blog post details.  
    - Inputs: From "Post Blog"  
    - Outputs: Feeds "If Scheduled".  

  - **If Scheduled**  
    - Type: If  
    - Role: Checks if the blog post is scheduled for future publishing to decide whether to update the sheet row.  
    - Inputs: From "Append row in sheet"  
    - Outputs: True → "Update row in sheet", False → no output.  

  - **Update row in sheet**  
    - Type: Google Sheets  
    - Role: Updates existing rows with scheduling details or status changes.  
    - Inputs: From "If Scheduled"  
    - Outputs: Feeds "Loop Over Items" (for batch continuation).  

  - **n8n | get wf**  
    - Type: n8n workflow node  
    - Role: Retrieves data or state from the current or another workflow for value replacement or update.  
    - Inputs: From "Write JSON"  
    - Outputs: Feeds "Replace values".  

  - **n8n | update**  
    - Type: n8n workflow node  
    - Role: Updates the workflow state or external system after replacement values are set.  
    - Inputs: From "Replace values"  
    - Outputs: Feeds back to "Loop Over Items" for processing next items.

---

### 3. Summary Table

| Node Name              | Node Type                            | Functional Role                                             | Input Node(s)              | Output Node(s)                     | Sticky Note                              |
|------------------------|------------------------------------|-------------------------------------------------------------|----------------------------|-----------------------------------|-----------------------------------------|
| Schedule Trigger       | Schedule Trigger                   | Starts workflow on schedule to process multiple clients     |                            | Get row(s) in sheet               |                                         |
| Get row(s) in sheet    | Google Sheets                     | Retrieves specific rows for processing                       | Schedule Trigger            | Loop Over Items                   |                                         |
| Loop Over Items        | Split In Batches                  | Processes rows in batches                                    | Get row(s) in sheet         | Map Data                         |                                         |
| Map Data               | Set                              | Maps input data fields into variables for prompts            | Loop Over Items, On form submission1 | Modify Prompt                   |                                         |
| Modify Prompt          | Code                             | Customizes AI prompt with client/blog data                   | Map Data                   | Client                          |                                         |
| Client                 | Google Sheets                    | Fetches detailed client info                                  | Modify Prompt              | AI Agent                       |                                         |
| AI Agent               | LangChain Agent                  | Manages AI content generation with memory                    | Google Gemini Chat Model, Simple Memory, Client | HTML to Elementor Format             | Retry on fail enabled                    |
| Google Gemini Chat Model | LangChain LM Chat Google Gemini | Calls Gemini AI for blog content                              | Client                     | AI Agent                       |                                         |
| Simple Memory          | LangChain Memory Buffer Window   | Maintains AI conversational context                          | AI Agent                   | AI Agent                       |                                         |
| HTML to Elementor Format | Code                             | Converts AI output to Elementor-compatible format             | AI Agent                   | WordpressAuth                  |                                         |
| WordpressAuth          | Code                             | Authenticates with WordPress                                 | HTML to Elementor Format   | Generate an image              |                                         |
| Generate an image      | LangChain Google Gemini          | AI generates blog-related images                              | WordpressAuth              | Upload Image                  |                                         |
| Upload Image           | HTTP Request                    | Uploads generated image to WordPress                          | Generate an image          | WordpressData                 |                                         |
| WordpressData          | Set                              | Prepares blog and image data for posting                     | Upload Image               | Post Blog                    |                                         |
| Post Blog              | HTTP Request                    | Posts blog content to WordPress                               | WordpressData              | Append row in sheet           |                                         |
| Append row in sheet    | Google Sheets                   | Logs published blog details                                   | Post Blog                  | If Scheduled                 |                                         |
| If Scheduled           | If                               | Checks if blog is scheduled for future publishing            | Append row in sheet        | Update row in sheet           |                                         |
| Update row in sheet    | Google Sheets                   | Updates sheet with scheduling/status info                    | If Scheduled               | Loop Over Items              |                                         |
| Google Sheets Trigger  | Google Sheets Trigger           | Triggers workflow on sheet changes                            |                            | Get all values               |                                         |
| Get all values         | Google Sheets                   | Retrieves all sheet data                                      | Google Sheets Trigger      | Format to 'values'           |                                         |
| Format to 'values'     | Set                              | Formats data for Google Sheets API                            | Get all values             | Write JSON                   |                                         |
| Write JSON             | Code                             | Prepares JSON for further processing                          | Format to 'values'         | n8n | get wf                 |                                         |
| n8n | get wf            | n8n workflow node               | Retrieves workflow data/state                                 | Write JSON                 | Replace values               |                                         |
| Replace values         | Set                              | Replaces placeholders in data                                 | n8n | get wf                  | n8n | update                   |                                         |
| n8n | update            | n8n workflow node               | Updates workflow state                                        | Replace values             | Loop Over Items              |                                         |
| On form submission1    | Form Trigger                   | Triggers workflow on form submission                          |                            | Map Data                     |                                         |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node:**  
   - Configure to run at desired intervals (e.g., daily).  
   - Connect output to "Get row(s) in sheet".

2. **Create a Google Sheets Trigger node:**  
   - Configure with the Google Sheet ID and worksheet to monitor changes.  
   - Connect output to "Get all values".

3. **Create a Google Sheets node "Get all values":**  
   - Set to retrieve all rows from the target sheet.  
   - Connect output to "Format to 'values'".

4. **Create a Set node "Format to 'values'":**  
   - Format the Google Sheets data into the structure expected downstream.  
   - Connect output to "Write JSON".

5. **Create a Code node "Write JSON":**  
   - Use JavaScript to prepare or log the JSON data.  
   - Connect output to "n8n | get wf".

6. **Create an n8n workflow node "n8n | get wf":**  
   - Retrieve external or workflow variables as needed.  
   - Connect output to "Replace values".

7. **Create a Set node "Replace values":**  
   - Replace placeholders or update values in data for Google Sheets update.  
   - Connect output to "n8n | update".

8. **Create an n8n workflow node "n8n | update":**  
   - Update workflow or external states.  
   - Connect output to "Loop Over Items".

9. **Create a Google Sheets node "Get row(s) in sheet":**  
   - Configure to retrieve rows scheduled for processing.  
   - Connect output to "Loop Over Items".

10. **Create a Split In Batches node "Loop Over Items":**  
    - Configure batch size for processing multiple clients/posts.  
    - Connect first output to "Map Data".

11. **Create a Set node "Map Data":**  
    - Map input data fields to variables suitable for prompt generation.  
    - Connect output to "Modify Prompt".

12. **Create a Code node "Modify Prompt":**  
    - Write JavaScript code to customize the AI prompt using mapped data.  
    - Connect output to "Client".

13. **Create a Google Sheets node "Client":**  
    - Retrieve client-specific info (e.g., SEO keywords).  
    - Connect output to "AI Agent".

14. **Create a LangChain LM Chat Google Gemini node "Google Gemini Chat Model":**  
    - Configure with Google Gemini credentials and model parameters.  
    - Connect output to "AI Agent" as language model input.

15. **Create a LangChain Agent node "AI Agent":**  
    - Enable retry on failure.  
    - Connect inputs from "Client" and "Google Gemini Chat Model".  
    - Connect output to "HTML to Elementor Format".

16. **Create a LangChain Memory Buffer Window node "Simple Memory":**  
    - Connect as AI memory for "AI Agent" to maintain context.

17. **Create a Code node "HTML to Elementor Format":**  
    - Write code to convert AI-generated HTML to Elementor-compatible format.  
    - Connect output to "WordpressAuth".

18. **Create a Code node "WordpressAuth":**  
    - Implement WordPress OAuth2 authentication or other methods.  
    - Connect output to "Generate an image".

19. **Create a LangChain Google Gemini node "Generate an image":**  
    - Configure for image generation with Gemini AI.  
    - Connect output to "Upload Image".

20. **Create an HTTP Request node "Upload Image":**  
    - Configure to upload images to WordPress media library endpoint.  
    - Connect output to "WordpressData".

21. **Create a Set node "WordpressData":**  
    - Prepare the blog post payload including formatted content and image URLs.  
    - Connect output to "Post Blog".

22. **Create an HTTP Request node "Post Blog":**  
    - Configure to send POST request to WordPress REST API for creating posts.  
    - Connect output to "Append row in sheet".

23. **Create a Google Sheets node "Append row in sheet":**  
    - Append new blog post details to tracking sheet.  
    - Connect output to "If Scheduled".

24. **Create an If node "If Scheduled":**  
    - Configure condition to check if post is scheduled for future publishing.  
    - True output connects to "Update row in sheet".

25. **Create a Google Sheets node "Update row in sheet":**  
    - Update scheduling status or metadata in Google Sheets.  
    - Connect output back to "Loop Over Items" for batch continuation.

26. **Create a Form Trigger node "On form submission1":**  
    - Configure webhook to receive blog requests via form.  
    - Connect output directly to "Map Data".

27. **Set up all credentials properly:**  
    - Google Sheets OAuth2 or API key.  
    - Google Gemini AI API credentials.  
    - WordPress OAuth2 credentials or API keys.  

---

### 5. General Notes & Resources

| Note Content                                                                 | Context or Link                                    |
|------------------------------------------------------------------------------|---------------------------------------------------|
| Workflow leverages Google Gemini AI for both text and image generation       | Requires Google Gemini API access                   |
| Elementor-compatible formatting ensures seamless integration with WordPress   | Elementor documentation: https://elementor.com/   |
| WordPress authentication must support OAuth2 or Application Passwords        | WordPress REST API docs: https://developer.wordpress.org/rest-api/ |
| Batch processing via "Loop Over Items" prevents API rate limit issues        | n8n docs on batching: https://docs.n8n.io/nodes/n8n-nodes-base.splitInBatches/ |
| Retry on fail enabled on AI Agent node to handle transient AI service errors  | Improves reliability in AI communication            |
| Workflow handles both scheduled batch processing and event-driven form inputs| Supports flexible blog post creation workflows      |

---

**Disclaimer:**  
The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data manipulated is legal and public.