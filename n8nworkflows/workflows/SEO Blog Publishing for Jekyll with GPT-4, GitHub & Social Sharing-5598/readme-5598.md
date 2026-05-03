SEO Blog Publishing for Jekyll with GPT-4, GitHub & Social Sharing

https://n8nworkflows.xyz/workflows/seo-blog-publishing-for-jekyll-with-gpt-4--github---social-sharing-5598


# SEO Blog Publishing for Jekyll with GPT-4, GitHub & Social Sharing

### 1. Workflow Overview

This workflow automates the creation, publishing, and social sharing of SEO-optimized blog posts for a Jekyll-based Italian recipe blog. It processes recipe data from a CSV file, uses GPT-4 powered AI to generate rich Markdown content tailored for SEO, commits the content to a GitHub repository to update the blog, and posts announcements on social media platforms. The workflow is structured into the following logical blocks:

- **1.1 Input Reception and Data Preparation:** Triggering the workflow, reading the CSV file containing recipes, extracting and batching recipe data for sequential processing.
- **1.2 AI Content Generation:** Using GPT-4 to write a detailed, SEO-optimized blog post in Markdown format based on each recipe’s metadata.
- **1.3 Markdown Construction and Commit:** Formatting the AI-generated content with Jekyll front-matter, generating slugified filenames, and committing the Markdown files to a GitHub repository.
- **1.4 Social Sharing (Optional/Disabled):** Posting about the newly published blog post to Twitter (X) and LinkedIn.
- **1.5 Housekeeping:** Removing processed lines from the CSV file to avoid duplication in future runs.
- **1.6 Scheduling:** Automating the workflow to run daily at 8 AM.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Data Preparation

**Overview:**  
This block triggers the workflow either manually or on a schedule, reads the CSV file containing recipe data, extracts structured information, and splits the data into batches for processing one recipe at a time.

**Nodes Involved:**  
- Start (manual trigger)  
- Schedule Trigger  
- Read CSV  
- Extract from File  
- Split In Batches  
- Remove the processed CSV line

**Node Details:**

- **Start**  
  - Type: Manual Trigger  
  - Role: Allows manual initiation of the workflow for testing or on-demand runs.  
  - Connections: Output connected to Read CSV.  
  - Edge Cases: None (user-controlled start).

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Automatically runs the workflow daily at 8:00 AM.  
  - Parameters: Triggers at hour 8 daily.  
  - Connections: Output connected to Read CSV.  
  - Edge Cases: Time zone considerations; ensure server time matches intended scheduling.

- **Read CSV**  
  - Type: Read Binary File  
  - Role: Reads the CSV file located at `/data/recipes.csv` containing all recipes awaiting publication. Data is read in binary form and stored in the `data` property.  
  - Parameters: File path set to `/data/recipes.csv`. Data property named `data`.  
  - Input: Trigger node(s).  
  - Output: Connected to Extract from File and Remove the processed csv line nodes.  
  - Edge Cases: File missing, access permission errors, CSV file too large.

- **Extract from File**  
  - Type: Extract From File  
  - Role: Parses the CSV data from the binary file using semicolon (`;`) delimiter into structured JSON for further processing.  
  - Parameters: Delimiter set to `;`.  
  - Input: Output from Read CSV.  
  - Output: Connected to Split In Batches.  
  - Edge Cases: CSV format errors, malformed lines.

- **Split In Batches**  
  - Type: Split In Batches  
  - Role: Processes one recipe at a time by splitting the dataset into batches of size 1, enabling sequential handling.  
  - Parameters: Batch size set to 1.  
  - Input: Output from Extract from File.  
  - Output: Connected to Copywriter AI Agent and Remove the processed csv line.  
  - Edge Cases: Empty input array, batch size errors.

- **Remove the processed CSV line**  
  - Type: Code  
  - Role: After successful processing of a batch, removes the corresponding recipe row from the CSV file to prevent republishing.  
  - Configuration: Reads `/data/recipes.csv`, filters out the row where the recipe title matches the processed item, and rewrites the CSV file if changed. Returns the original JSON item.  
  - Input: Output from Split In Batches.  
  - Output: None (end of this branch).  
  - Edge Cases: File write permissions, concurrent file access, partial processing errors.

---

#### 2.2 AI Content Generation

**Overview:**  
This block uses two AI-related nodes in a chain where a smaller GPT-4 model triggers a specialized AI agent to generate a fully structured, SEO-optimized Italian recipe blog post in Markdown.

**Nodes Involved:**  
- gpt-4o-mini (OpenAI GPT-4o-mini)  
- Copywriter AI Agent (Langchain AI agent)

**Node Details:**

- **gpt-4o-mini**  
  - Type: OpenAI Chat Language Model (gpt-4o-mini)  
  - Role: Acts as a base language model to provide input to the Copywriter AI Agent.  
  - Parameters: Uses the "gpt-4o-mini" model from OpenAI with default options.  
  - Credentials: OpenAI API credentials required.  
  - Input: Triggered from Split In Batches.  
  - Output: Connected to Copywriter AI Agent’s language model input.  
  - Edge Cases: API rate limits, network errors, invalid credentials.

- **Copywriter AI Agent**  
  - Type: Langchain AI Agent  
  - Role: Core AI node that writes an SEO-first Italian food blog article based on the incoming recipe data.  
  - Parameters:  
    - Detailed prompt guides the AI to write an 800-1000 word article with specific SEO rules, structure, and tone.  
    - Input variables include recipe title, description, main and secondary keywords.  
    - Output is strictly Markdown content with no front matter or explanations.  
  - Input: Receives language model output from gpt-4o-mini and JSON data from Split In Batches.  
  - Output: Connects to the Code node for markdown construction.  
  - Edge Cases: AI content generation failures, prompt parsing errors, token limits.

---

#### 2.3 Markdown Construction and Commit

**Overview:**  
This block prepares the AI-generated content for Jekyll by adding front matter and slugified file paths and commits the Markdown file to a GitHub repository.

**Nodes Involved:**  
- Code  
- Wait Until Publish (disabled)  
- Commit Markdown

**Node Details:**

- **Code**  
  - Type: Code (JavaScript)  
  - Role:  
    - Takes the AI output and recipe metadata.  
    - Generates an SEO-friendly slug from the recipe title.  
    - Constructs Jekyll front matter with title, date, and layout.  
    - Combines front matter and AI Markdown content into a single Markdown string.  
    - Sets the Markdown file path inside `_posts` per Jekyll conventions.  
  - Key Expressions: Uses regex to slugify the title; builds front matter with recipe date and title.  
  - Input: Output from Copywriter AI Agent (Markdown content) plus original JSON from batch.  
  - Output: Connected to Wait Until Publish node.  
  - Edge Cases: Title with special characters causing slug errors, date format issues.

- **Wait Until Publish**  
  - Type: Wait (disabled)  
  - Role: Intended to delay publishing until a specified publication date/time.  
  - Parameters: Waits until the date/time specified in the `data_pubblicazione` field.  
  - Input: Output from Code node.  
  - Output: Connected to Commit Markdown node.  
  - Status: Disabled (not active in current workflow).  
  - Edge Cases: Time zone mismatches, date format parsing errors.

- **Commit Markdown**  
  - Type: GitHub  
  - Role: Commits the generated Markdown file to the `jekyll-recipe-ai` repository on GitHub under the user `antonio-backend-projects`.  
  - Parameters:  
    - Repository and owner specified.  
    - File path derived from the `markdownPath` field.  
    - File content is the Markdown string from Code node.  
    - Commit message includes the recipe title dynamically.  
  - Credentials: GitHub API credentials required.  
  - Input: Output from Wait Until Publish (or Code if Wait is disabled).  
  - Output: Connected to Post on X node.  
  - Edge Cases: Commit conflicts, API rate limits, invalid credentials, file path errors.

---

#### 2.4 Social Sharing (Optional/Disabled)

**Overview:**  
This block is designed to share the newly published blog post on social media platforms Twitter (X) and LinkedIn. Both nodes are currently disabled.

**Nodes Involved:**  
- Post on X (Twitter)  
- Post on LinkedIn

**Node Details:**

- **Post on X**  
  - Type: Twitter  
  - Role: Posts a tweet announcing the new blog post.  
  - Parameters: Operation set to create a tweet. Content is not explicitly configured in JSON.  
  - Input: Output from Commit Markdown.  
  - Output: Connected to Post on LinkedIn.  
  - Status: Disabled.  
  - Edge Cases: Twitter API limits, authentication errors, content formatting issues.

- **Post on LinkedIn**  
  - Type: LinkedIn  
  - Role: Posts an update on LinkedIn with the blog post title as text.  
  - Parameters: Text dynamically set to recipe title.  
  - Input: Output from Post on X.  
  - Status: Disabled.  
  - Edge Cases: LinkedIn API errors, authentication failures.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                              | Input Node(s)              | Output Node(s)               | Sticky Note                                                 |
|-------------------------|----------------------------------|----------------------------------------------|----------------------------|-----------------------------|-------------------------------------------------------------|
| Start                   | Manual Trigger                   | Manual start of workflow                      | —                          | Read CSV                    |                                                             |
| Schedule Trigger        | Schedule Trigger                 | Automated daily trigger at 8 AM               | —                          | Read CSV                    |                                                             |
| Read CSV                | Read Binary File                 | Reads CSV file with recipes                   | Start, Schedule Trigger     | Extract from File, Remove the processed csv line |                                                             |
| Extract from File       | Extract From File                | Parses CSV data into JSON                      | Read CSV                   | Split In Batches            |                                                             |
| Split In Batches        | Split In Batches                 | Processes recipes one at a time                | Extract from File           | Copywriter AI Agent, Remove the processed csv line |                                                             |
| Remove the processed csv line | Code                      | Removes published recipe line from CSV        | Split In Batches            | —                           |                                                             |
| gpt-4o-mini             | OpenAI Language Model            | Provides base language model input             | Split In Batches            | Copywriter AI Agent (lmChatOpenAi input) |                                                             |
| Copywriter AI Agent     | Langchain AI Agent               | Generates SEO-optimized blog post Markdown    | Split In Batches, gpt-4o-mini | Code                      |                                                             |
| Code                    | Code                            | Builds Jekyll front matter and full Markdown  | Copywriter AI Agent         | Wait Until Publish           |                                                             |
| Wait Until Publish      | Wait                           | Delays publishing until specified date        | Code                       | Commit Markdown             | Disabled currently                                          |
| Commit Markdown        | GitHub                          | Commits Markdown file to GitHub repository    | Wait Until Publish          | Post on X                   |                                                             |
| Post on X               | Twitter                         | Tweets about the new blog post                  | Commit Markdown             | Post on LinkedIn            | Disabled currently                                          |
| Post on LinkedIn       | LinkedIn                        | Shares post on LinkedIn                          | Post on X                  | —                           | Disabled currently                                          |
| Sticky Note4            | Sticky Note                     | Notes on SEO optimized blog post writing       | —                          | —                           | ## Write SEO Optimized Blog Post                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create manual trigger node named "Start".**

2. **Create a "Schedule Trigger" node:**  
   - Set to run daily at 8:00 AM.

3. **Connect "Start" and "Schedule Trigger" both to "Read CSV" node:**  
   - Type: Read Binary File  
   - Parameters: File path `/data/recipes.csv`, data property named `data`.

4. **Add "Extract from File" node:**  
   - Type: Extract From File  
   - Parameters: Delimiter set to `;`  
   - Connect the output of "Read CSV" to this node.

5. **Add "Split In Batches" node:**  
   - Parameters: Batch size = 1  
   - Connect output of "Extract from File" to this node.

6. **Add "gpt-4o-mini" node:**  
   - Type: OpenAI Language Model (Chat)  
   - Model: `gpt-4o-mini`  
   - Credentials: Configure OpenAI API credentials.  
   - Connect output of "Split In Batches" to this node.

7. **Add "Copywriter AI Agent" node:**  
   - Type: Langchain AI Agent  
   - Parameters: Copy the detailed prompt specifying SEO-first Italian recipe blog post generation with placeholders for title, description, keywords, and strict Markdown-only output.  
   - Connect the output of "Split In Batches" to the agent’s JSON input, and the output of "gpt-4o-mini" to the AI language model input of this node.

8. **Add "Code" node:**  
   - Type: Code (JavaScript)  
   - Paste the script that:  
     - Reads the first batch item JSON.  
     - Creates SEO-friendly slug from title.  
     - Builds Jekyll front matter with title, date, layout.  
     - Concatenates front matter and AI Markdown content.  
     - Sets Markdown file path `_posts/YYYY-MM-DD-slug.md`.  
   - Connect output of "Copywriter AI Agent" to this node.

9. **Add "Wait Until Publish" node (optional, currently disabled):**  
   - Type: Wait  
   - Parameters: Wait until date/time from `data_pubblicazione` field.  
   - Connect output of "Code" node to this node.

10. **Add "Commit Markdown" node:**  
    - Type: GitHub  
    - Repository owner: `antonio-backend-projects`  
    - Repository name: `jekyll-recipe-ai`  
    - File path: dynamic from `markdownPath`  
    - File content: from `markdownContent`  
    - Commit message: `Add recipe: <recipe title>`  
    - Credentials: Configure GitHub API credentials.  
    - Connect output of "Wait Until Publish" (or "Code" if wait disabled) to this node.

11. **Add social sharing nodes (optional, disabled):**  
    - "Post on X" (Twitter) node: operation to create tweet. Connect output of "Commit Markdown" to this node.  
    - "Post on LinkedIn" node: post with text from recipe title. Connect output of "Post on X" to this node.  
    - Keep these nodes disabled unless social sharing is desired.

12. **Add "Remove the processed csv line" node:**  
    - Type: Code (JavaScript)  
    - Script to read `/data/recipes.csv`, filter out the row corresponding to the processed recipe title, rewrite the CSV file if changed.  
    - Connect output of "Split In Batches" also to this node (parallel branch).

---

### 5. General Notes & Resources

| Note Content                                                                                           | Context or Link                                                                                                  |
|------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| The workflow generates SEO-optimized blog articles in Italian specifically tailored for Jekyll blogs. | Workflow description and prompt embedded in "Copywriter AI Agent" node.                                         |
| Disabled social sharing nodes suggest optional extension for Twitter and LinkedIn integration.        | Nodes "Post on X" and "Post on LinkedIn" are disabled by default, indicating optional use.                      |
| The CSV file `/data/recipes.csv` must be accessible with proper file permissions for read/write.     | Critical for reading input data and removing processed lines, file path must be configured correctly.           |
| GitHub repository `jekyll-recipe-ai` must exist with proper access rights for commit operations.     | GitHub credentials must have repo write permissions.                                                            |
| The slug generation code uses regex to create SEO-friendly URLs; special characters must be sanitized. | See "Code" node for slug generation details.                                                                     |
| Wait Until Publish node is disabled, indicating immediate publishing; it can be enabled to schedule posts. | Pay attention to date/time format and time zone settings if enabling.                                            |
| Workflow uses OpenAI’s GPT-4o-mini and Langchain AI agent package; ensure compatible n8n version.     | Langchain agent node requires n8n version supporting version 1.8+ of this node type.                             |

---

**Disclaimer:** The provided text originates exclusively from an automated workflow created with n8n, an integration and automation tool. All content respects applicable content policies, contains no illegal or offensive material, and handles only legal and public data.