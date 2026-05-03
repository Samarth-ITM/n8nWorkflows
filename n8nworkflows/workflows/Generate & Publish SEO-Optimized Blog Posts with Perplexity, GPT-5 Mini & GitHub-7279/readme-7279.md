Generate & Publish SEO-Optimized Blog Posts with Perplexity, GPT-5 Mini & GitHub

https://n8nworkflows.xyz/workflows/generate---publish-seo-optimized-blog-posts-with-perplexity--gpt-5-mini---github-7279


# Generate & Publish SEO-Optimized Blog Posts with Perplexity, GPT-5 Mini & GitHub

### 1. Workflow Overview

This workflow automates the generation and publication of SEO-optimized blog posts by leveraging AI-powered research and writing tools, then publishing the content directly to a GitHub repository. It targets content creators and digital marketers who want to automate SEO content creation and deployment workflows.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Topic Discovery:** Periodically triggers the workflow every 8 hours to find trending blog topics using Perplexity AI.
- **1.2 Topic Preparation & Deep Research:** Sets the selected topic data and performs deeper research on it with Perplexity AI to gather rich information.
- **1.3 AI Content Generation:** Uses OpenAI (GPT-5 Mini) to generate a detailed article (2000+ words) based on the research.
- **1.4 Content Formatting:** Formats the AI-generated article into a structured JSON suitable for publication.
- **1.5 Publishing:** Creates a new file in a GitHub repository containing the SEO-optimized blog post.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Topic Discovery

- **Overview:**  
  This block initiates the workflow on a fixed schedule (every 8 hours) and uses Perplexity AI to identify trending topics suitable for blog content generation.

- **Nodes Involved:**  
  - Every 8 hours (Schedule Trigger)  
  - Find Trending Topics (Perplexity)

- **Node Details:**

  - **Every 8 hours:**  
    - Type: Schedule Trigger  
    - Configuration: Triggers the workflow every 8 hours without additional filters.  
    - Input: None (trigger node)  
    - Output: Connected to "Find Trending Topics"  
    - Edge Cases: System time changes or downtime could delay triggers.

  - **Find Trending Topics:**  
    - Type: Perplexity Node (AI Research)  
    - Configuration: Queries Perplexity AI to retrieve currently trending blog topics relevant for SEO.  
    - Input: Triggered by "Every 8 hours" node.  
    - Output: Passes data to "Set Topic Data" node.  
    - Edge Cases: API rate limits, network errors, or invalid responses from Perplexity.

#### 2.2 Topic Preparation & Deep Research

- **Overview:**  
  This block sets the topic context and performs a deeper, more detailed research phase on the selected trending topic.

- **Nodes Involved:**  
  - Set Topic Data (Set)  
  - Deep Research (Perplexity)

- **Node Details:**

  - **Set Topic Data:**  
    - Type: Set Node  
    - Configuration: Initializes and structures data variables related to the selected topic from the previous node.  
    - Input: From "Find Trending Topics"  
    - Output: Feeds into "Deep Research"  
    - Edge Cases: Data structure mismatches or missing topic data.

  - **Deep Research:**  
    - Type: Perplexity Node (AI Research)  
    - Configuration: Performs an in-depth AI-driven query on the set topic to gather detailed content, statistics, and insights.  
    - Input: From "Set Topic Data"  
    - Output: Feeds into "Generate 2000+ Word Article"  
    - Edge Cases: API errors, long response times, incomplete data.

#### 2.3 AI Content Generation

- **Overview:**  
  Generates a comprehensive 2000+ word article using OpenAI's GPT-5 Mini, based on the research data.

- **Nodes Involved:**  
  - Generate 2000+ Word Article (OpenAI)

- **Node Details:**

  - **Generate 2000+ Word Article:**  
    - Type: OpenAI Node  
    - Configuration: Uses GPT-5 Mini to create a detailed blog article, optimized for SEO and readability.  
    - Input: From "Deep Research"  
    - Output: Passes generated text to "Format Blog JSON"  
    - Key Variables: Prompt likely includes researched content context.  
    - Edge Cases: Token limits, rate limits, generation errors, incomplete outputs.

#### 2.4 Content Formatting

- **Overview:**  
  Transforms the raw AI-generated article into a structured JSON format suitable for GitHub file creation and further processing.

- **Nodes Involved:**  
  - Format Blog JSON (Code)

- **Node Details:**

  - **Format Blog JSON:**  
    - Type: Code Node (JavaScript)  
    - Configuration: Custom script parses and organizes the article content into JSON format, possibly including metadata such as title, date, tags, and body.  
    - Input: From "Generate 2000+ Word Article"  
    - Output: Connected to "Create File in GitHub"  
    - Edge Cases: Script errors, malformed content, missing fields.

#### 2.5 Publishing

- **Overview:**  
  Publishes the SEO-optimized blog post by creating a new file in the specified GitHub repository.

- **Nodes Involved:**  
  - Create File in GitHub (GitHub Node)

- **Node Details:**

  - **Create File in GitHub:**  
    - Type: GitHub Node  
    - Configuration: Commits a new file containing the blog post JSON to a GitHub repository (likely with a timestamped filename).  
    - Input: From "Format Blog JSON"  
    - Output: End of workflow (no further nodes)  
    - Edge Cases: Authentication errors, permission issues, filename conflicts, API rate limits.

---

### 3. Summary Table

| Node Name              | Node Type           | Functional Role                    | Input Node(s)           | Output Node(s)             | Sticky Note                |
|------------------------|---------------------|----------------------------------|------------------------|----------------------------|----------------------------|
| Every 8 hours          | Schedule Trigger    | Triggers workflow every 8 hours  | None                   | Find Trending Topics       |                            |
| Find Trending Topics   | Perplexity AI       | Identifies trending blog topics  | Every 8 hours          | Set Topic Data             |                            |
| Set Topic Data         | Set Node            | Sets and structures topic data   | Find Trending Topics   | Deep Research              |                            |
| Deep Research          | Perplexity AI       | Performs detailed topic research | Set Topic Data         | Generate 2000+ Word Article|                            |
| Generate 2000+ Word Article | OpenAI Node      | Generates SEO-optimized article  | Deep Research          | Format Blog JSON           |                            |
| Format Blog JSON       | Code Node           | Formats article into JSON         | Generate 2000+ Word Article | Create File in GitHub   |                            |
| Create File in GitHub  | GitHub Node         | Publishes article to GitHub repo | Format Blog JSON       | None                      |                            |
| Sticky Note            | Sticky Note         | (Empty)                          | None                   | None                      |                            |
| Sticky Note 2          | Sticky Note         | (Empty)                          | None                   | None                      |                            |
| Sticky Note 3          | Sticky Note         | (Empty)                          | None                   | None                      |                            |
| Sticky Note 4          | Sticky Note         | (Empty)                          | None                   | None                      |                            |
| Sticky Note 5          | Sticky Note         | (Empty)                          | None                   | None                      |                            |
| Sticky Note 6          | Sticky Note         | (Empty)                          | None                   | None                      |                            |
| Sticky Note 7          | Sticky Note         | (Empty)                          | None                   | None                      |                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow** named "AI Blog Generator - Auto-publish SEO content to GitHub".

2. **Add a Schedule Trigger node**:  
   - Name: `Every 8 hours`  
   - Type: Schedule Trigger  
   - Parameters: Set to trigger every 8 hours (e.g., repeat interval = 8 hours).

3. **Add a Perplexity node** to find trending topics:  
   - Name: `Find Trending Topics`  
   - Connect input from `Every 8 hours` node.  
   - Configure API credentials for Perplexity AI.  
   - Set query to retrieve trending blog topics (customize prompt as needed).

4. **Add a Set node** to prepare topic data:  
   - Name: `Set Topic Data`  
   - Connect input from `Find Trending Topics`.  
   - Define variables to extract and store topic title, keywords, or any relevant metadata.

5. **Add another Perplexity node** for deeper research:  
   - Name: `Deep Research`  
   - Connect input from `Set Topic Data`.  
   - Configure to perform detailed queries based on the topic data.  
   - Use appropriate API credentials.

6. **Add an OpenAI node** for article generation:  
   - Name: `Generate 2000+ Word Article`  
   - Connect input from `Deep Research`.  
   - Configure API credentials for OpenAI (GPT-5 Mini model).  
   - Set prompt to generate a detailed SEO-optimized article of 2000+ words based on the research data.

7. **Add a Code node** to format the article JSON:  
   - Name: `Format Blog JSON`  
   - Connect input from `Generate 2000+ Word Article`.  
   - Implement JavaScript code to parse the generated article and structure it into JSON with fields such as title, date, tags, and content body.

8. **Add a GitHub node** to create a file:  
   - Name: `Create File in GitHub`  
   - Connect input from `Format Blog JSON`.  
   - Configure GitHub OAuth2 credentials with write permissions.  
   - Set repository, branch, file path (e.g., `/posts/YYYY-MM-DD-topic-title.md`), and content from the formatted JSON.  
   - Commit message can be dynamically set to include the article title or date.

9. **Verify all connections** flow sequentially:  
   `Every 8 hours` → `Find Trending Topics` → `Set Topic Data` → `Deep Research` → `Generate 2000+ Word Article` → `Format Blog JSON` → `Create File in GitHub`.

10. **Test the workflow manually** and adjust parameters as necessary, checking API limits, authentication, and output formatting.

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                   |
|-------------------------------------------------------------------------------------------------|--------------------------------------------------|
| Workflow uses Perplexity AI and OpenAI GPT-5 Mini for research and content generation.           | Requires valid API credentials for both services |
| GitHub node requires OAuth2 credentials with repo write access for publishing blog posts.       | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-base.github/ |
| Recommended to monitor API rate limits and handle errors gracefully in production workflows.    | n8n error handling documentation                  |
| The workflow is designed for automated SEO content generation with minimal manual intervention. |                                                  |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.