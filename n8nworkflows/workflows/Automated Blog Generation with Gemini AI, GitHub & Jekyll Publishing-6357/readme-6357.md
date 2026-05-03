Automated Blog Generation with Gemini AI, GitHub & Jekyll Publishing

https://n8nworkflows.xyz/workflows/automated-blog-generation-with-gemini-ai--github---jekyll-publishing-6357


# Automated Blog Generation with Gemini AI, GitHub & Jekyll Publishing

### 1. Workflow Overview

This workflow automates the creation and publishing of a technical blog post. It is designed to:

- Retrieve blog topics from a Google Sheet that tracks blog ideas and their status.
- Perform deep research on the selected topic using web search (via Tavily) and Wikipedia content.
- Summarize and format the research context.
- Use the Google Gemini AI language model (or equivalent LLM) to generate a complete, well-structured blog post in Markdown suitable for Jekyll static site generation.
- Prepare and commit the generated Markdown file to a GitHub repository hosting a Jekyll site.
- Update the blog topic status in the Google Sheet to “done” upon successful publishing.

The workflow is organized into the following logical blocks:

- **1.1 Trigger and Input Retrieval:** Scheduling and fetching blog topics from Google Sheets.
- **1.2 Research and Context Gathering:** Using Tavily web search and Wikipedia to collect background information.
- **1.3 Content Generation:** Formatting research results and generating the blog post with Gemini AI.
- **1.4 File Preparation and Publishing:** Formatting the Markdown file and committing it to GitHub.
- **1.5 Status Update:** Marking the blog topic as done in the Google Sheet.

---

### 2. Block-by-Block Analysis

#### 1.1 Trigger and Input Retrieval

**Overview:**  
This block initiates the workflow on a schedule, fetches a blog topic from a Google Sheet that is not yet marked as done, and extracts the relevant topic text for processing.

**Nodes Involved:**  
- Schedule Trigger  
- Get Topic from Google Sheet  
- Extract Blog Title

**Node Details:**

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Triggers workflow execution daily at 09:00 (configured hour).  
  - Configuration: Set to trigger at 9 AM every day.  
  - Inputs: None (trigger node).  
  - Outputs: Connects to “Get Topic from Google Sheet”.  
  - Edge Cases: Misconfiguration could cause missed triggers; time zone considerations apply (Asia/Kolkata set).  

- **Get Topic from Google Sheet**  
  - Type: Google Sheets  
  - Role: Reads entries from the specified sheet filtered by status not done (status column filter).  
  - Configuration: Reads from sheet named “blog list” (gid=0) in a Google Sheets document. Filters rows where status is not “done”.  
  - Inputs: From Schedule Trigger.  
  - Outputs: Passes data to “Extract Blog Title”.  
  - Credentials: Google Sheets OAuth2.  
  - Edge Cases: Google API quota limits, connectivity issues, empty or malformed rows.  

- **Extract Blog Title**  
  - Type: Set  
  - Role: Extracts the blog topic title field from the sheet data and sets it as a variable `topic`.  
  - Configuration: Assigns `topic` from `$json.Title` (assumed sheet column).  
  - Inputs: From “Get Topic from Google Sheet”.  
  - Outputs: Connects to “Tavily Web Search” for research.  
  - Edge Cases: Missing or empty Title field would cause downstream nodes to fail or generate irrelevant content.

---

#### 1.2 Research and Context Gathering

**Overview:**  
This block performs deep research on the blog topic by querying Tavily web search and Wikipedia. It aggregates and summarizes the collected information to build context for the AI generation.

**Nodes Involved:**  
- Tavily Web Search  
- Fetch Wikipedia Context  
- Summarize Search Results

**Node Details:**

- **Tavily Web Search**  
  - Type: Tavily Search Node  
  - Role: Performs an advanced web search on the extracted topic to gather relevant articles and summaries.  
  - Configuration: Query set to the extracted `topic` variable, `search_depth` set to “advanced” for richer results.  
  - Inputs: From “Extract Blog Title”.  
  - Outputs: Connects to “Summarize Search Results” and “Google Gemini Chat Model” (via ai_languageModel interface).  
  - Credentials: Tavily API key.  
  - Edge Cases: API rate limits, query failures, zero results.  

- **Fetch Wikipedia Context**  
  - Type: Wikipedia Tool Node (Langchain)  
  - Role: Retrieves Wikipedia content and images related to the topic for enriched context.  
  - Configuration: No parameters explicitly set, uses topic context dynamically.  
  - Inputs: AI Tool input from Tavily results.  
  - Outputs: Feeds into AI agent for blog generation.  
  - Edge Cases: Topic not found on Wikipedia, disambiguation pages, API errors.  

- **Summarize Search Results**  
  - Type: Code Node (JavaScript)  
  - Role: Processes Tavily search results, picks top 3, formats with titles, snippets, and sources into a Markdown-like context string.  
  - Configuration: Custom JS code that creates a concatenated summary with headings and links.  
  - Inputs: From “Tavily Web Search”.  
  - Outputs: Passes summary context and topic to “Generate Blog with Gemini” node.  
  - Edge Cases: Empty search results, malformed data.  

---

#### 1.3 Content Generation

**Overview:**  
This block uses a Gemini AI language model agent to generate a full technical blog post in Markdown format. The prompt includes strict formatting instructions for Jekyll compatibility.

**Nodes Involved:**  
- Generate Blog with Gemini

**Node Details:**

- **Generate Blog with Gemini**  
  - Type: Langchain Agent Node (Google Gemini AI)  
  - Role: Receives the topic and summarized context; generates a complete Markdown blog post including YAML front matter, table of contents, sections, code snippets, images, and links per detailed prompt instructions.  
  - Configuration:  
    - Prompt text dynamically includes topic and context.  
    - System message sets strict formatting rules for Markdown suitable for Jekyll.  
    - Uses Google Gemini 2.5 Pro model via Google Palm API credentials.  
  - Inputs: From “Summarize Search Results” and “Fetch Wikipedia Context” (via AI tool interface).  
  - Outputs: Produces fully formatted Markdown content.  
  - Credentials: Google Gemini API (Google Palm API).  
  - Edge Cases: API timeouts, rate limits, malformed input causing generation errors, prompt misconfiguration.  

---

#### 1.4 File Preparation and Publishing

**Overview:**  
This block formats the generated Markdown content into a file structure, constructs a commit message, and uploads the file to a specified GitHub repository.

**Nodes Involved:**  
- Prepare File for Commit  
- Commit Blog Post to GitHub

**Node Details:**

- **Prepare File for Commit**  
  - Type: Set  
  - Role: Sets file path, content, and commit message for GitHub upload.  
  - Configuration:  
    - Filename format: `YYYY-MM-DD-topic-slug.md` (topic slug is lowercase and spaces replaced by hyphens).  
    - File content: Markdown output from AI agent.  
    - Commit message: “Add blog post: [Topic Title]”.  
  - Inputs: From “Generate Blog with Gemini”.  
  - Outputs: Connects to “Commit Blog Post to GitHub”.  
  - Edge Cases: Incorrect topic formatting causing invalid filenames, empty content.  

- **Commit Blog Post to GitHub**  
  - Type: GitHub Node  
  - Role: Commits the Markdown file to the specified GitHub repository and path (typically `_posts/` folder for Jekyll).  
  - Configuration:  
    - Repository owner and name configured.  
    - File path set dynamically (e.g., `_posts/YYYY-MM-DD-topic-slug.md`).  
    - Commit message as prepared.  
    - Authentication via OAuth2 token.  
  - Inputs: From “Prepare File for Commit”.  
  - Outputs: Connects to “Mark Topic as Done in Sheet”.  
  - Credentials: GitHub OAuth2 credentials.  
  - Edge Cases: GitHub API errors, auth failures, branch conflicts, rate limits.  

---

#### 1.5 Status Update

**Overview:**  
Marks the processed blog topic as done in the Google Sheet to prevent reprocessing.

**Nodes Involved:**  
- Mark Topic as Done in Sheet

**Node Details:**

- **Mark Topic as Done in Sheet**  
  - Type: Google Sheets  
  - Role: Updates the row corresponding to the blog topic with status “done”.  
  - Configuration:  
    - Matches row by `row_number`.  
    - Sets `status` column to “done”.  
    - Uses same sheet and document as input node.  
  - Inputs: From “Commit Blog Post to GitHub”.  
  - Credentials: Google Sheets OAuth2.  
  - Edge Cases: Row mismatch causing no update, API limits, concurrency conflicts.

---

### 3. Summary Table

| Node Name                  | Node Type                                | Functional Role                            | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                           |
|----------------------------|-----------------------------------------|--------------------------------------------|------------------------------|------------------------------|-----------------------------------------------------------------------------------------------------|
| Schedule Trigger           | Schedule Trigger                        | Start workflow on schedule                  | None                         | Get Topic from Google Sheet   | ## Trigger node<br>Configure the node for your preference or else you can use other option as well. |
| Get Topic from Google Sheet | Google Sheets                          | Retrieve blog topics not marked “done”     | Schedule Trigger             | Extract Blog Title            | ## Get Topic from spreadsheet<br>This will capture the topic which is not done yet.                  |
| Extract Blog Title          | Set                                   | Extract topic text from sheet data          | Get Topic from Google Sheet  | Tavily Web Search             | ## Extract topic<br>This will extract topic text from the sheet node.                               |
| Tavily Web Search           | Tavily Search                         | Perform web search for topic research       | Extract Blog Title           | Summarize Search Results      | ## Deep Search about topic<br>We have a system node that gathers information about a given topic.    |
| Fetch Wikipedia Context     | Wikipedia Tool (Langchain)             | Retrieve Wikipedia content for topic        | AI Tool input from Tavily    | Generate Blog with Gemini     |                                                                                                     |
| Summarize Search Results    | Code                                  | Summarize and format Tavily search results  | Tavily Web Search            | Generate Blog with Gemini     | ## Format the result<br>Extracts and formats search details into a predefined structure.             |
| Generate Blog with Gemini   | Langchain AI Agent (Google Gemini)     | Generate full Markdown blog post             | Summarize Search Results, Wikipedia Tool | Prepare File for Commit        | ## Ai agent<br>Standardized prompt generating well-structured markdown blog post.                    |
| Prepare File for Commit     | Set                                   | Prepare filename, content, commit message   | Generate Blog with Gemini    | Commit Blog Post to GitHub    | ## Set the file for github upload<br>Prepares content for seamless GitHub integration.              |
| Commit Blog Post to GitHub  | GitHub                                | Commit Markdown blog post to GitHub repo    | Prepare File for Commit      | Mark Topic as Done in Sheet   | ## Github upload<br>Uploads the blog post to the GitHub repository.                                 |
| Mark Topic as Done in Sheet | Google Sheets                         | Update blog status to “done” in Google Sheet| Commit Blog Post to GitHub   | None                         | ## Change status<br>Marks the blog topic status in the sheet as done after upload.                  |
| Sticky Note                | Sticky Note                           | Descriptive notes                            | None                         | None                         | ## Automated Blog Creation description and instructions.                                           |
| Sticky Note1               | Sticky Note                           | Trigger node comment                         | None                         | None                         | ## Trigger node configuration note.                                                                |
| Sticky Note2               | Sticky Note                           | Google Sheets topic retrieval comment       | None                         | None                         | ## Get Topic from spreadsheet comment.                                                             |
| Sticky Note3               | Sticky Note                           | Extract topic comment                        | None                         | None                         | ## Extract topic comment.                                                                           |
| Sticky Note4               | Sticky Note                           | Deep search explanation                      | None                         | None                         | ## Deep Search about topic comment.                                                                |
| Sticky Note5               | Sticky Note                           | Formatting search results explanation        | None                         | None                         | ## Format the result comment.                                                                       |
| Sticky Note6               | Sticky Note                           | AI agent comment                             | None                         | None                         | ## Ai agent comment.                                                                                |
| Sticky Note7               | Sticky Note                           | File preparation comment                     | None                         | None                         | ## Set the file for github upload comment.                                                         |
| Sticky Note8               | Sticky Note                           | GitHub upload comment                        | None                         | None                         | ## Github upload comment.                                                                           |
| Sticky Note9               | Sticky Note                           | Status update comment                        | None                         | None                         | ## Change status comment.                                                                           |
| Sticky Note10              | Sticky Note                           | Setup and integration instructions          | None                         | None                         | # n8n Integrations & GitHub Pages Jekyll Setup with prerequisites and stepwise instructions.       |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger node**  
   - Set trigger time to daily at 09:00 (adjust timezone as needed).  
   - No credentials required.

2. **Add a Google Sheets node “Get Topic from Google Sheet”**  
   - Operation: Read rows  
   - Sheet: `gid=0` of your Google Sheet document (containing blog topics).  
   - Filter: Only rows where `status` column is not “done”.  
   - Connect input from Schedule Trigger.  
   - Configure Google Sheets OAuth2 credentials.

3. **Add a Set node “Extract Blog Title”**  
   - Assign a new variable `topic` set to the field containing the blog title from the sheet (e.g., `{{$json.Title}}`).  
   - Connect input from “Get Topic from Google Sheet”.

4. **Add a Tavily Web Search node**  
   - Configure query to use `{{$json.topic}}`.  
   - Set search depth to “advanced” for richer results.  
   - Connect input from “Extract Blog Title”.  
   - Add Tavily API credentials.

5. **Add a Code node “Summarize Search Results”**  
   - Use JS code to select top 3 Tavily results, format with title, snippet, and source URL into a Markdown context string.  
   - Input from Tavily Web Search node.

6. **Add a Wikipedia Tool node**  
   - Set to fetch Wikipedia content dynamically based on topic.  
   - Connect input from Tavily node via AI tool interface or directly integrate with Langchain AI agent.

7. **Add a Langchain Agent node “Generate Blog with Gemini”**  
   - Use Google Gemini 2.5 Pro model with your Google Palm API credentials.  
   - Set prompt to generate a complete Markdown blog post with strict Jekyll front matter and formatting rules.  
   - Inject variables `topic` and `context` from “Summarize Search Results” and Wikipedia content.  
   - Connect inputs accordingly.

8. **Add a Set node “Prepare File for Commit”**  
   - Create filename as `YYYY-MM-DD-topic-slug.md` where topic slug is lowercased and spaces replaced by hyphens.  
   - Assign `fileContent` to the AI-generated Markdown output.  
   - Compose a commit message like `Add blog post: [Topic Title]`.  
   - Input from AI agent output.

9. **Add a GitHub node “Commit Blog Post to GitHub”**  
   - Configure repository owner and name.  
   - Set file path to `_posts/{{filePath}}`.  
   - Use commit message and file content from previous node.  
   - Use OAuth2 credentials for GitHub.  
   - Connect input from “Prepare File for Commit”.

10. **Add a Google Sheets node “Mark Topic as Done in Sheet”**  
    - Update operation on the same Google Sheet.  
    - Match row by `row_number` from input.  
    - Update `status` column to “done”.  
    - Input from GitHub commit node.  
    - Use Google Sheets OAuth2 credentials.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                      |
|--------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Automated blog creation workflow that integrates LLM (Google Gemini), Tavily web search, Wikipedia, Google Sheets, and GitHub publishing. | Workflow description sticky note in the original workflow.                                         |
| Instructions for setting up Tavily API key: Sign up at https://www.tavily.com/, get API key from dashboard.                               | Sticky note with setup instructions for prerequisites.                                             |
| Google Sheets, GitHub OAuth2, and Google Gemini credentials must be configured in n8n with appropriate permissions.                       | Credential setup reminders from sticky notes and workflow nodes.                                   |
| Jekyll blog posts require Markdown files with YAML front matter in `_posts/` folder - filename format: `YYYY-MM-DD-title.md`.             | Formatting rules embedded in AI prompt and GitHub commit node configuration.                       |
| For GitHub Pages + Jekyll, ensure repository is configured correctly with Pages enabled and `_config.yml` set for your blog site.          | Setup instructions in sticky notes and GitHub publishing node context.                             |
| The workflow uses advanced prompt engineering to ensure the blog post is ready to publish without further manual editing.                  | Prompt details in the Gemini AI node configuration.                                               |

---

This documentation enables advanced users and automation agents to understand, reproduce, and maintain the “Automated Blog Generation with Gemini AI, GitHub & Jekyll Publishing” workflow in n8n. It covers all nodes, logical flow, integration points, and potential failure scenarios.