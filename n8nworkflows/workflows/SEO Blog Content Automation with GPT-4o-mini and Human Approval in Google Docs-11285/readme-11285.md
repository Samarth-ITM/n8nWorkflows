SEO Blog Content Automation with GPT-4o-mini and Human Approval in Google Docs

https://n8nworkflows.xyz/workflows/seo-blog-content-automation-with-gpt-4o-mini-and-human-approval-in-google-docs-11285


# SEO Blog Content Automation with GPT-4o-mini and Human Approval in Google Docs

### 1. Workflow Overview

This workflow automates the creation, review, and publication of SEO-optimized blog content using GPT-4o-mini AI models, integrated with Google Sheets for topic management and Google Docs for content storage. It is tailored for content teams or marketers who want to streamline blog production with AI assistance and human approval. The workflow is composed of several logical blocks:

- **1.1 Topic Intake and Tracking:** Capturing blog topic suggestions via a form and recording them in Google Sheets as a content tracker.
- **1.2 AI Blog Content Generation:** Using an AI agent powered by GPT-4o-mini to improve the initial topic and generate a full SEO-optimized blog post.
- **1.3 Human Approval Process:** Sending the generated content for human review and collecting approval or feedback via a Gmail form.
- **1.4 Content Revision:** If needed, the AI revises the blog post based on the human feedback.
- **1.5 Final Content Publishing:** Creating a Google Docs document with the finalized blog content and updating the Google Sheets tracker with the publication status and document link.

---

### 2. Block-by-Block Analysis

#### 2.1 Topic Intake and Tracking

**Overview:**  
This block captures new blog topics submitted via a form and appends them to a Google Sheets tracker for processing.

**Nodes Involved:**  
- On form submission  
- Append row in sheet  
- Get Topic from Google Sheets

**Node Details:**  

- **On form submission**  
  - **Type:** Form Trigger  
  - **Role:** Receives blog topic suggestions and reference URLs from users through a web form.  
  - **Configuration:**  
    - Form titled "Suggest the Topic for Blog"  
    - Fields: "Topic" (required), "Reference link" (optional)  
    - Button labeled "Create Content"  
  - **Inputs:** None (trigger)  
  - **Outputs:** Form submission data with topic and reference link  
  - **Edge cases:** Missing required fields, incorrect URL format

- **Append row in sheet**  
  - **Type:** Google Sheets (append operation)  
  - **Role:** Adds the submitted topic and reference URL as a new row in the Google Sheets tracker.  
  - **Configuration:**  
    - Spreadsheet ID and sheet "Content" configured  
    - Columns: Topic, Referenc URL  
  - **Inputs:** Data from form submission  
  - **Outputs:** Confirmation of row appended  
  - **Edge cases:** Google Sheets API rate limits, permission errors

- **Get Topic from Google Sheets**  
  - **Type:** Google Sheets (read operation)  
  - **Role:** Retrieves topics from the tracker with specific filters (e.g., status) to pass to the AI agent for content generation.  
  - **Configuration:**  
    - Filters on "Status" column, presumably to get topics marked as new or pending  
    - Sheet and document IDs aligned with the tracker  
  - **Inputs:** Output from Append row in sheet  
  - **Outputs:** Topic data including Topic, Reference URL, row number, and status  
  - **Edge cases:** Empty results if no topics match filter, API errors

---

#### 2.2 AI Blog Content Generation

**Overview:**  
This block uses an AI agent (GPT-4o-mini) that acts as an expert SEO copywriter. It improves the blog topic title and writes a full-length SEO-optimized blog post in markdown format.

**Nodes Involved:**  
- Simple Memory  
- OpenAI Chat Model  
- Copywriter AI Agent  
- Structured Output Parser  
- Set Data

**Node Details:**  

- **Simple Memory**  
  - **Type:** LangChain memory buffer  
  - **Role:** Maintains conversation context for the AI based on the topic title, enabling better continuity in AI responses.  
  - **Configuration:**  
    - Session key is set dynamically using the topic from Google Sheets  
    - Context window of 10 interactions  
  - **Inputs:** Topic from Google Sheets  
  - **Outputs:** Contextualized memory for AI agent  
  - **Edge cases:** Memory overflow or loss if session key mismatches

- **OpenAI Chat Model**  
  - **Type:** LangChain OpenAI chat model wrapper  
  - **Role:** Provides GPT-4o-mini language model for text generation  
  - **Configuration:**  
    - Model: gpt-4o-mini  
    - OpenAI API credentials configured  
  - **Inputs:** Prompts from AI agent nodes  
  - **Outputs:** AI-generated text responses  
  - **Edge cases:** API rate limits, timeout, auth failures

- **Copywriter AI Agent**  
  - **Type:** LangChain Agent node  
  - **Role:** Acts as an expert SEO copywriter to improve topic title and write a full blog post  
  - **Configuration:**  
    - Custom prompt defining task: improve title, write 800-1200 words SEO-friendly blog post with headings, bullet points, strong intro and conclusion  
    - Input variables: topic title and reference URL from Google Sheets  
    - Output format: JSON with "title" and "content" in markdown  
    - Uses OpenAI Chat Model and Simple Memory as language model and memory backend  
    - Uses Structured Output Parser to extract JSON from AI response  
  - **Inputs:** Topic from Google Sheets, memory, OpenAI model  
  - **Outputs:** JSON with improved title and blog content  
  - **Edge cases:** Parsing errors if AI output is malformed, incomplete response

- **Structured Output Parser**  
  - **Type:** LangChain output parser  
  - **Role:** Parses AI agent output into structured JSON format as per the schema example (title, content)  
  - **Inputs:** Raw AI output from Copywriter AI Agent  
  - **Outputs:** Parsed JSON with title and content  
  - **Edge cases:** Parsing failures if AI output deviates from expected format

- **Set Data**  
  - **Type:** Set node  
  - **Role:** Extracts parsed title and content and sets them as separate variables for downstream use  
  - **Configuration:**  
    - Sets "Topic Title" to parsed title  
    - Sets "Content" to parsed blog content  
  - **Inputs:** Parsed AI output  
  - **Outputs:** Data with clean title and content fields  
  - **Edge cases:** Missing fields if parsing failed

---

#### 2.3 Human Approval Process

**Overview:**  
This block sends the generated blog content to a human reviewer for approval, collects feedback, and branches accordingly based on approval status.

**Nodes Involved:**  
- Send Content for Approval  
- Approval Result (Switch node)

**Node Details:**  

- **Send Content for Approval**  
  - **Type:** Gmail node (send and wait)  
  - **Role:** Sends an email containing the generated blog title and content for human approval, and waits for a response via a custom form embedded in the email.  
  - **Configuration:**  
    - Recipient email configured (placeholder "<your email>" to be replaced)  
    - Subject: "Approval Required for Blog Content"  
    - Message body includes generated title and content variables  
    - Form fields include dropdown for "Approve Content?" with options Yes/No/Cancel and a textarea for "Content Feedback"  
    - Waits for user response (custom form response)  
    - Gmail OAuth2 credentials configured  
  - **Inputs:** Data from Set Data node (title and content)  
  - **Outputs:** Form response data with approval decision and feedback  
  - **Edge cases:** Email delivery failures, no response timeout, invalid feedback input

- **Approval Result**  
  - **Type:** Switch node  
  - **Role:** Routes workflow execution based on approval decision: Yes, No, or Cancel.  
  - **Configuration:**  
    - Checks "Approve Content?" field in form response  
    - Outputs renamed to "Yes", "No", and "Cancel" branches accordingly  
  - **Inputs:** Form response data from Send Content for Approval  
  - **Outputs:**  
    - "Yes" branch: proceeds to publish content  
    - "No" branch: triggers content revision  
    - "Cancel" branch: marks topic as canceled in tracker  
  - **Edge cases:** Unexpected or missing approval values

---

#### 2.4 Content Revision

**Overview:**  
If the content is not approved, this block uses an AI agent to revise the blog post based on the human reviewer’s feedback.

**Nodes Involved:**  
- Copywriter Revision Agent  
- Structured Output Parser  
- Set Data

**Node Details:**  

- **Copywriter Revision Agent**  
  - **Type:** LangChain Agent node  
  - **Role:** Expert copywriter AI that revises the existing blog post applying user feedback, maintaining tone and structure.  
  - **Configuration:**  
    - Prompt includes original topic title, user feedback, and original content  
    - Output expected in structured JSON (title and content)  
    - Uses OpenAI Chat Model and Simple Memory for language model and context  
    - Uses Structured Output Parser for output parsing  
  - **Inputs:**  
    - Topic from Google Sheets  
    - User feedback from approval form  
    - Original AI-generated content  
  - **Outputs:** Revised blog post JSON  
  - **Edge cases:** Parsing errors, incomplete feedback, AI misunderstanding feedback

- **Structured Output Parser**  
  - (Same as above, shared with original AI agent)  

- **Set Data**  
  - (Same as above, prepares revised data for next steps)  

---

#### 2.5 Final Content Publishing

**Overview:**  
This block creates a Google Docs document with the approved blog content and updates the Google Sheets tracker with the document link and publication date.

**Nodes Involved:**  
- Update Topic Status on Google Sheets  
- Create Blog file  
- Add blog content in file  
- Update sheet with blog post link

**Node Details:**  

- **Update Topic Status on Google Sheets**  
  - **Type:** Google Sheets (update operation)  
  - **Role:** Updates the blog topic row to mark status as "Approved" (or other relevant status) after approval.  
  - **Configuration:**  
    - Columns updated: Title (improved title), Status ("Approved")  
    - Uses row_number from Google Sheets to target correct row  
    - Spreadsheet and sheet settings consistent with tracker  
  - **Inputs:** Data from Set Data node or Copywriter Revision Agent output  
  - **Outputs:** Confirmation of update  
  - **Edge cases:** Race conditions if row_number incorrect, API errors

- **Create Blog file**  
  - **Type:** Google Docs (create document)  
  - **Role:** Creates a new Google Docs document with the improved blog title as the document title in a specified folder.  
  - **Configuration:**  
    - Document title set to improved topic title  
    - Folder ID specified for storage  
    - Credentials: Google Docs OAuth2  
  - **Inputs:** Title from Set Data node  
  - **Outputs:** Google Docs document metadata including document ID  
  - **Edge cases:** Permission errors, quota limits

- **Add blog content in file**  
  - **Type:** Google Docs (update document)  
  - **Role:** Inserts the full blog content into the newly created Google Docs document.  
  - **Configuration:**  
    - Operation: update (insert text)  
    - Document URL from previous node’s output (document ID)  
    - Text content from Set Data node  
  - **Inputs:** Document ID from Create Blog file, content from Set Data  
  - **Outputs:** Updated document metadata  
  - **Edge cases:** Document lock issues, API failures

- **Update sheet with blog post link**  
  - **Type:** Google Sheets (update operation)  
  - **Role:** Updates the tracker row with the published date (current timestamp) and Google Docs link for the blog post.  
  - **Configuration:**  
    - Columns updated: Published (timestamp), Link to document (Google Docs URL), row_number for matching  
    - Spreadsheet and sheet consistent with tracker  
  - **Inputs:** Document metadata from Add blog content in file, current timestamp  
  - **Outputs:** Confirmation of update  
  - **Edge cases:** Timestamp format errors, wrong linkage if row_number mismatched

---

### 3. Summary Table

| Node Name                    | Node Type                          | Functional Role                        | Input Node(s)                         | Output Node(s)                        | Sticky Note                                           |
|------------------------------|----------------------------------|-------------------------------------|-------------------------------------|-------------------------------------|-------------------------------------------------------|
| On form submission            | Form Trigger                     | Captures blog topic submission form | -                                   | Append row in sheet                  |                                                       |
| Append row in sheet           | Google Sheets (append)           | Adds new topic to Google Sheets     | On form submission                  | Get Topic from Google Sheets         |                                                       |
| Get Topic from Google Sheets  | Google Sheets (read)             | Retrieves topics for processing     | Append row in sheet                 | Copywriter AI Agent                  |                                                       |
| Simple Memory                | LangChain Memory Buffer          | Maintains AI conversation context   | Get Topic from Google Sheets         | Copywriter AI Agent, Copywriter Revision Agent |                                                       |
| OpenAI Chat Model            | LangChain OpenAI Chat Model      | Provides GPT-4o-mini language model | Copywriter AI Agent, Copywriter Revision Agent | Copywriter AI Agent, Copywriter Revision Agent |                                                       |
| Copywriter AI Agent          | LangChain Agent                  | Generates improved title & blog post | Get Topic from Google Sheets, Simple Memory, OpenAI Chat Model | Set Data                           | "Create the Blog" - AI writes SEO-optimized blog posts (800-1200 words). Tip: Suggest adding Humanizer & AI detector steps after AI output. |
| Structured Output Parser     | LangChain Output Parser          | Parses AI JSON output                | Copywriter AI Agent, Copywriter Revision Agent | Copywriter AI Agent, Copywriter Revision Agent |                                                       |
| Set Data                    | Set node                        | Extracts and sets title and content | Copywriter AI Agent, Copywriter Revision Agent | Send Content for Approval, Update Topic Status on Google Sheets |                                                       |
| Send Content for Approval    | Gmail node (send and wait)       | Sends blog for human approval       | Set Data                           | Approval Result                     | "Approval Step" - Get an approval and make modifications |
| Approval Result              | Switch node                     | Routes workflow based on approval   | Send Content for Approval          | Update Topic Status, Copywriter Revision Agent |                                                       |
| Copywriter Revision Agent    | LangChain Agent                  | Revises blog post based on feedback | Approval Result (No branch), Simple Memory, OpenAI Chat Model | Set Data                           | "Take Feedback and improve" - AI improves content based on human feedback. |
| Update Topic Status on Google Sheets | Google Sheets (update)           | Updates tracker status and title    | Approval Result (Yes, Cancel), Set Data | Create Blog file, Update Topic Status on Google Sheets | "Update Sheet post approval" - Update Status and add content in sheet |
| Create Blog file             | Google Docs (create)             | Creates Google Docs document        | Update Topic Status on Google Sheets | Add blog content in file            |                                                       |
| Add blog content in file     | Google Docs (update)             | Inserts blog content into document  | Create Blog file                  | Update sheet with blog post link    |                                                       |
| Update sheet with blog post link | Google Sheets (update)           | Updates tracker with published date and doc link | Add blog content in file           | -                                   |                                                       |
| Sticky Note1                | Sticky Note                     | Notes - Identify Topic               | -                                   | -                                   | "Identify Topic: Identify topics for blogs and update in sheet" |
| Sticky Note3                | Sticky Note                     | Notes - Create the Blog              | -                                   | -                                   | "Create the Blog: AI writes SEO-optimized blog posts (800-1200 words). Tip: Suggest adding Humanizer & AI detector steps after the AI agent output." |
| Sticky Note4                | Sticky Note                     | Notes - Approval Step                | -                                   | -                                   | "Approval Step: Get an approval and make modifications" |
| Sticky Note5                | Sticky Note                     | Notes - Take Feedback and improve   | -                                   | -                                   | "Take Feedback and improve: AI improves content based on human feedback." |
| Sticky Note6                | Sticky Note                     | Notes - Update Sheet post approval  | -                                   | -                                   | "Update Sheet post approval: Update Status and add the content in the sheet" |
| Sticky Note (Main)          | Sticky Note                     | Workflow overview and setup notes   | -                                   | -                                   | "**How it works**: Through a form it captures the topic for blog. Added to sheet/tracker. AI writes blog. Human approval & revision. Blogs saved as document & link in tracker. Setup instructions included. Tip: Add AI humanizer and GPTZero check." |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger:**  
   - Node: On form submission  
   - Type: Form Trigger  
   - Form title: "Suggest the Topic for Blog"  
   - Fields:  
     - "Topic" (string, required)  
     - "Reference link" (string)  
   - Button label: "Create Content"  
   - This node triggers the workflow when a user submits a blog topic.

2. **Append Topic to Google Sheets:**  
   - Node: Append row in sheet  
   - Type: Google Sheets (Append)  
   - Spreadsheet: Set your Google Sheets document ID and sheet named "Content"  
   - Columns mapped:  
     - Topic → from form field "Topic"  
     - Referenc URL → from form field "Reference link"  
   - Use Google Sheets OAuth2 credentials.

3. **Retrieve Topic for Processing:**  
   - Node: Get Topic from Google Sheets  
   - Type: Google Sheets (Read)  
   - Sheet and document same as above  
   - Filter rows by "Status" column for new or pending topics (set filter as needed)  
   - Use same credentials.

4. **Setup AI Simple Memory:**  
   - Node: Simple Memory  
   - Type: LangChain Memory Buffer Window  
   - Session key: Set dynamically to the topic string from Google Sheets  
   - Context window length: 10 (to hold recent context)  

5. **Configure OpenAI Chat Model:**  
   - Node: OpenAI Chat Model  
   - Type: LangChain LM Chat OpenAI  
   - Model: Select "gpt-4o-mini"  
   - Credentials: Provide OpenAI API key credentials

6. **Create Copywriter AI Agent:**  
   - Node: Copywriter AI Agent  
   - Type: LangChain Agent  
   - Prompt: Define a detailed prompt instructing the AI to:  
     - Improve the topic title for SEO  
     - Write an 800-1200 word SEO-optimized blog post with markdown formatting (headings, bullet points, strong intro & conclusion)  
     - Output JSON with "title" and "content"  
   - Variables in prompt:  
     - Topic from Google Sheets  
     - Reference URL from Google Sheets  
   - Set language model: OpenAI Chat Model  
   - Set memory: Simple Memory  

7. **Add Structured Output Parser:**  
   - Node: Structured Output Parser  
   - Type: LangChain Output Parser Structured  
   - JSON Schema Example:  
     ```json
     {
       "title": "Improved SEO-Optimized Title",
       "content": "Full blog post content in markdown format"
     }
     ```  
   - Connect it to the Copywriter AI Agent output to parse JSON.

8. **Set Data Variables:**  
   - Node: Set Data  
   - Type: Set  
   - Map parsed JSON fields:  
     - "Topic Title" = parsed "title"  
     - "Content" = parsed "content"  

9. **Send Content for Approval:**  
   - Node: Send Content for Approval  
   - Type: Gmail node (send and wait)  
   - Recipient: Set the email of the content reviewer  
   - Subject: "Approval Required for Blog Content"  
   - Message body: Include generated "Topic Title" and "Content"  
   - Form fields for response:  
     - Dropdown: "Approve Content?" options Yes, No, Cancel (required)  
     - Textarea: "Content Feedback"  
   - Use Gmail OAuth2 credentials.

10. **Add Approval Result Switch:**  
    - Node: Approval Result  
    - Type: Switch  
    - Check the "Approve Content?" response field  
    - Branches:  
      - Yes → Proceed to update sheet status and publishing  
      - No → Trigger Copywriter Revision Agent for content improvement  
      - Cancel → Update status as canceled

11. **Create Copywriter Revision Agent:**  
    - Node: Copywriter Revision Agent  
    - Type: LangChain Agent  
    - Prompt: Instruct AI to revise blog post based on:  
      - Original topic title  
      - User's feedback from approval form  
      - Original blog content  
    - Output JSON with updated title and content  
    - Use same OpenAI Chat Model and Simple Memory  
    - Connect Structured Output Parser and Set Data nodes for processing revised output.

12. **Update Topic Status in Google Sheets (Approval or Cancel):**  
    - Node: Update Topic Status on Google Sheets  
    - Type: Google Sheets (Update)  
    - Update row matching by row_number  
    - Set columns:  
      - Title = improved/revised title  
      - Status = "Approved" or "Canceled" based on approval  
    - Use Google Sheets OAuth2 credentials.

13. **Create Google Docs Document:**  
    - Node: Create Blog file  
    - Type: Google Docs (Create document)  
    - Title: Use "Topic Title" from Set Data  
    - Folder ID: Set target folder for storing blog docs  
    - Use Google Docs OAuth2 credentials.

14. **Insert Blog Content into Document:**  
    - Node: Add blog content in file  
    - Type: Google Docs (Update document)  
    - Document URL: Use document ID from Create Blog file  
    - Action: Insert text with blog "Content" from Set Data

15. **Update Sheet with Blog Post Link and Publish Date:**  
    - Node: Update sheet with blog post link  
    - Type: Google Sheets (Update)  
    - Update row by row_number  
    - Set columns:  
      - Published = current timestamp  
      - Link to document = URL of Google Doc created  
    - Use Google Sheets OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                            | Context or Link                                                                                                  |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| Main workflow captures blog topics via form, uses AI to generate and revise content based on human approval, then saves final blogs in Google Docs.    | Sticky Note (Main)                                                                                               |
| Suggestion to integrate AI humanizer and AI content detector tools such as GPTZero to improve content authenticity and quality before sending for approval. | Sticky Note Main and Sticky Note3                                                                               |
| Ensure Gmail OAuth2 and Google APIs credentials are properly configured with the required permissions for Sheets, Docs, and Gmail access.               | General setup requirement                                                                                        |
| Google Sheets tracker sheet should have columns: Topic, Referenc URL, Title, Link to document, Status, Published, row_number.                           | Workflow description and setup notes                                                                             |
| Replace placeholder email "<your email>" in Send Content for Approval node with the actual reviewer email address.                                       | Send Content for Approval node configuration                                                                     |
| Folder ID in Create Blog file node must be set to the Google Drive folder intended for storing blog documents.                                           | Create Blog file node configuration                                                                              |

---

This completes the comprehensive analysis and documentation of the "SEO Blog Content Automation with GPT-4o-mini and Human Approval in Google Docs" workflow. It provides detailed insights, node configurations, and instructions for reproduction, ensuring ease of modification and troubleshooting.