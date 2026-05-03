YouTube to Multilingual Blogs: Convert Videos to SEO-Friendly Google Doc Articles

https://n8nworkflows.xyz/workflows/youtube-to-multilingual-blogs--convert-videos-to-seo-friendly-google-doc-articles-5671


# YouTube to Multilingual Blogs: Convert Videos to SEO-Friendly Google Doc Articles

---

### 1. Workflow Overview

This workflow automates the conversion of a YouTube video into a multilingual, SEO-friendly blog article that is directly inserted into a Google Docs document. It is designed for content creators, marketers, and bloggers aiming to repurpose video content efficiently into written format across multiple languages.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Captures user input via a web form — YouTube video URL and target language.
- **1.2 Data Mapping:** Prepares the input data for API consumption by mapping form fields to workflow variables.
- **1.3 AI Blog Generation:** Calls an external API to convert the video content into a well-structured blog post in the selected language.
- **1.4 Document Insertion:** Appends the generated blog content into a specified Google Docs document for editing, publishing, or archiving.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  This block triggers the workflow upon user submission of a form containing the YouTube video URL and the desired language for the blog post. It serves as the entry point.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**

  - **Node:** On form submission  
    - Type: `formTrigger`  
    - Role: Captures input data via a web form.  
    - Configuration:  
      - Form titled "YouTube Video To Blog" with two fields:  
        - "Video URL" (text input, required)  
        - "language" (dropdown with English, Hindi, French, German, Gujarati; required)  
      - Webhook ID assigned for external form submissions.  
    - Expressions/Variables: None at this stage; outputs form fields as JSON.  
    - Input: External HTTP trigger from form submission.  
    - Output: JSON object containing "Video URL" and "language" fields.  
    - Edge Cases / Failure Modes:  
      - Missing required fields leads to no trigger or error on submission.  
      - Invalid URL formats or unsupported languages are not validated here and may cause downstream failures.  
    - Version Requirements: n8n version supporting `formTrigger` node with version 2.2 or later.

#### 1.2 Data Mapping

- **Overview:**  
  Transforms the raw form input into a structured format suitable for the subsequent API call, ensuring variable names and values are correctly assigned.

- **Nodes Involved:**  
  - Mapper

- **Node Details:**

  - **Node:** Mapper  
    - Type: `set` node  
    - Role: Assigns workflow variables from incoming JSON data.  
    - Configuration:  
      - Assigns "Video URL" from form field `"Video URL"`  
      - Assigns "language" from form field `"language"`  
    - Expressions:  
      - `={{ $json["Video URL"] }}`  
      - `={{ $json.language }}`  
    - Input: Output of "On form submission" node.  
    - Output: JSON with mapped fields for API consumption.  
    - Edge Cases:  
      - If inputs are missing or malformed, variables may be empty, causing API call to fail.  
    - Version: Supports expression language version 3.4 or later.

#### 1.3 AI Blog Generation

- **Overview:**  
  Sends the mapped video URL and language to an external AI-powered API that generates a blog post based on the video content.

- **Nodes Involved:**  
  - YouTube to Blog

- **Node Details:**

  - **Node:** YouTube to Blog  
    - Type: `httpRequest`  
    - Role: Calls external API at `youtube-to-blog.p.rapidapi.com` to convert video to blog content.  
    - Configuration:  
      - Method: POST  
      - URL: `https://youtube-to-blog.p.rapidapi.com/ytblogs/index.php`  
      - Body Parameters:  
        - `videoUrl`: dynamically set from `"Video URL"`  
        - `language`: dynamically set from `"language"`  
      - Headers:  
        - `x-rapidapi-host`: `youtube-to-blog.p.rapidapi.com`  
        - `x-rapidapi-key`: User must provide their RapidAPI key (credential required)  
      - Sends both body and headers in request.  
    - Expressions:  
      - `={{ $json["Video URL"] }}`  
      - `={{ $json.language }}`  
    - Input: Mapped data from "Mapper" node.  
    - Output: API response containing generated blog content, expected as plain text or JSON field `data`.  
    - Edge Cases / Failure Modes:  
      - API key missing or invalid: authentication errors.  
      - API downtime or rate limiting: request failures or timeouts.  
      - Invalid video URLs or unsupported languages: API may return errors or empty content.  
      - Network issues causing request failure.  
    - Version: Uses n8n HTTP Request node version 4.2 or later for enhanced HTTP features.

#### 1.4 Document Insertion

- **Overview:**  
  Inserts the generated blog content into a pre-configured Google Docs document using a service account for authentication.

- **Nodes Involved:**  
  - Google Docs

- **Node Details:**

  - **Node:** Google Docs  
    - Type: `googleDocs`  
    - Role: Updates a Google Docs document by appending the blog content.  
    - Configuration:  
      - Operation: `update` with action to insert text.  
      - Document URL: Static URL of the target Google Docs document.  
      - Text to insert: Dynamically set to the API response field `data` from "YouTube to Blog" node.  
      - Authentication: Uses a Google Service Account credential.  
    - Expressions: `'={{ $json.data }}'` where `$json.data` is API response content.  
    - Input: Blog content from "YouTube to Blog".  
    - Output: Google Docs update confirmation or error.  
    - Edge Cases / Failure Modes:  
      - Authentication failure due to invalid or expired service account credentials.  
      - Document URL inaccessible or permissions insufficient.  
      - Large content exceeding Google Docs API limits.  
      - Network or API errors during update.  
    - Version: Requires n8n Google Docs node version 2 or later with service account support.

---

### 3. Summary Table

| Node Name          | Node Type       | Functional Role                                  | Input Node(s)       | Output Node(s)     | Sticky Note                                                                                          |
|--------------------|-----------------|-------------------------------------------------|---------------------|--------------------|----------------------------------------------------------------------------------------------------|
| On form submission  | formTrigger     | Receives YouTube URL and language via form      | (External trigger)  | Mapper             | Triggers when a user submits the form with a YouTube video URL and a selected language.            |
| Mapper             | set             | Maps form input fields to workflow variables    | On form submission  | YouTube to Blog    | Maps the form input fields (`Video URL`, `language`) to variables for use in the next steps.       |
| YouTube to Blog     | httpRequest     | Calls external API to generate blog content     | Mapper              | Google Docs        | Sends the video URL and language to the external API (`youtube-to-blog.p.rapidapi.com`).            |
| Google Docs        | googleDocs       | Inserts generated blog content into Google Doc  | YouTube to Blog     | (Workflow end)     | Inserts the generated blog content into a predefined Google Docs document for storage and access.  |
| Sticky Note        | stickyNote      | Documentation and overview                       | (No input)          | (No output)        | # 📽️ YouTube Video to Blog – n8n Workflow (Full detailed overview and benefits)                    |
| Sticky Note1       | stickyNote      | Workflow node summary                            | (No input)          | (No output)        | ## 🔄 Workflow Node Overview – YouTube Video to Blog (Node function summaries)                      |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Trigger Node:**  
   - Add a `formTrigger` node named "On form submission".  
   - Configure form: Title = "YouTube Video To Blog".  
   - Add required fields:  
     - Text field labeled "Video URL" (required).  
     - Dropdown field labeled "language" with options: English, Hindi, French, German, Gujarati (required).  
   - Save and note the webhook URL generated.

2. **Add a Set Node for Data Mapping:**  
   - Add a `set` node named "Mapper".  
   - Configure to assign two variables:  
     - "Video URL" with value expression `={{ $json["Video URL"] }}`  
     - "language" with value expression `={{ $json.language }}`  
   - Connect output of "On form submission" to "Mapper".

3. **Add HTTP Request Node for Blog Generation:**  
   - Add an `httpRequest` node named "YouTube to Blog".  
   - Configure:  
     - HTTP Method: POST  
     - URL: `https://youtube-to-blog.p.rapidapi.com/ytblogs/index.php`  
     - Send Body: true with parameters:  
       - `videoUrl`: `={{ $json["Video URL"] }}`  
       - `language`: `={{ $json.language }}`  
     - Send Headers: true with:  
       - `x-rapidapi-host`: `youtube-to-blog.p.rapidapi.com`  
       - `x-rapidapi-key`: [Your RapidAPI key here]  
   - Connect output of "Mapper" node to "YouTube to Blog".

4. **Add Google Docs Node to Insert Blog Content:**  
   - Add a `googleDocs` node named "Google Docs".  
   - Configure:  
     - Operation: `update`  
     - Action: Insert text.  
     - Document URL: Paste your Google Docs document URL (e.g., `https://docs.google.com/document/d/1ZC52bgdGhpe9clkJrjnTBFq5JfIgqjVNryO1E0FwNG8/edit`)  
     - Text to insert: `={{ $json.data }}` (content from API response)  
     - Authentication: Choose or create a Google API credential using a Service Account with access to the document.  
   - Connect output of "YouTube to Blog" to "Google Docs".

5. **Activate and Test the Workflow:**  
   - Save all nodes and activate the workflow.  
   - Submit the form through the webhook URL with a valid YouTube video URL and select a language.  
   - Confirm that the blog content appears in the specified Google Docs document.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                  | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| This workflow automates YouTube video to blog conversion with AI-generated content in multiple languages, saving time and manual effort.                                    | Workflow purpose and benefits as detailed in Sticky Note.                                                      |
| Potential enhancements include CMS auto-publishing, email or Slack integration for blog summaries, SEO tools integration, and blog archiving via Google Sheets.             | Workflow future development ideas.                                                                              |
| Requires a valid RapidAPI key for `youtube-to-blog.p.rapidapi.com` API. Ensure quota and access permissions are handled.                                                    | API authentication requirement.                                                                                  |
| Google Docs node must use a Service Account credential with editor permissions for the target document.                                                                       | Google API credential setup recommendation.                                                                      |
| Workflow tested with n8n version supporting nodes: formTrigger v2.2, set v3.4, httpRequest v4.2, googleDocs v2.                                                               | Version compatibility note.                                                                                       |
| For more detailed understanding and overview, refer to the in-workflow sticky notes documenting each step and node purpose.                                                | Sticky Note content in the workflow JSON.                                                                        |

---

**Disclaimer:**  
The text provided derives solely from an automated workflow created using n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected material. All processed data is legal and publicly accessible.