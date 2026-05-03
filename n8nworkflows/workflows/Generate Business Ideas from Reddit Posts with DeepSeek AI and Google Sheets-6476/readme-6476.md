Generate Business Ideas from Reddit Posts with DeepSeek AI and Google Sheets

https://n8nworkflows.xyz/workflows/generate-business-ideas-from-reddit-posts-with-deepseek-ai-and-google-sheets-6476


# Generate Business Ideas from Reddit Posts with DeepSeek AI and Google Sheets

### 1. Workflow Overview

This workflow automates the generation of business ideas from Reddit posts using AI processing and stores the results in Google Sheets. It targets users seeking to extract innovative business concepts from social media content, leveraging natural language processing and summarization.

Logical blocks:

- **1.1 Input Reception:** Triggers workflow execution and fetches Reddit posts.
- **1.2 Filtering & Validation:** Filters relevant posts and determines if they describe a business problem.
- **1.3 Summarization:** Summarizes filtered Reddit posts to concise content.
- **1.4 Idea Generation:** Uses AI to generate business ideas based on summaries.
- **1.5 Output & Storage:** Processes final data and appends or updates rows in Google Sheets.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Initiates the workflow either manually or via another workflow and retrieves multiple Reddit posts for analysis.
- **Nodes Involved:** 
  - When Executed by Another Workflow
  - Get many posts
  - If

- **Node Details:**

  - **When Executed by Another Workflow**
    - Type: Execute Workflow Trigger
    - Role: Listens for execution from an external workflow, allowing integration or batch processing.
    - Configuration: Default trigger parameters.
    - Inputs: None (trigger node).
    - Outputs: Connected to "Get many posts".
    - Edge Cases: Trigger may fail if upstream workflow is misconfigured or lacks permissions.

  - **Get many posts (Reddit)**
    - Type: Reddit Node
    - Role: Fetches a batch of Reddit posts for processing.
    - Configuration: Likely set to retrieve posts from specific subreddits or queries (parameters not shown).
    - Inputs: Trigger from "When Executed by Another Workflow" or manual trigger.
    - Outputs: Connected to "If" node.
    - Edge Cases: Reddit API rate limits, authentication errors, empty result sets.

  - **If**
    - Type: Conditional Node
    - Role: Evaluates if fetched Reddit posts meet criteria to continue processing.
    - Configuration: Conditions not explicitly defined; possibly checks if posts exist or contain relevant content.
    - Inputs: From "Get many posts".
    - Outputs: Passes positive branch to "Edit Fields".
    - Edge Cases: Condition misconfiguration can block valid posts or allow irrelevant data.

---

#### 1.2 Filtering & Validation

- **Overview:** Refines and prepares Reddit posts for AI evaluation, and determines if the posts describe a business problem.
- **Nodes Involved:** 
  - Edit Fields
  - Merge
  - Is it a Business Problem?
  - OpenRouter Chat Model

- **Node Details:**

  - **Edit Fields**
    - Type: Set Node
    - Role: Modifies or enriches data fields in the Reddit posts for downstream processing.
    - Configuration: Likely sets or transforms fields to prepare data for AI chains.
    - Inputs: From "If" node.
    - Outputs: Connected to "Merge" and "Is it a Business Problem?" nodes.
    - Edge Cases: Incorrect field manipulation can break expressions downstream.

  - **Merge**
    - Type: Merge Node
    - Role: Combines data streams, possibly merging original posts with AI validation results.
    - Configuration: Default parameters; likely merging based on index or key.
    - Inputs: From "Edit Fields" and "Is it a Business Problem?".
    - Outputs: To "Filter".
    - Edge Cases: Mismatch in data arrays can cause merge failures or data loss.

  - **Is it a Business Problem?**
    - Type: LangChain LLM Chain
    - Role: AI chain that classifies whether the Reddit post describes a business problem.
    - Configuration: Uses OpenRouter Chat Model for natural language understanding.
    - Inputs: From "Edit Fields" (via OpenRouter Chat Model node).
    - Outputs: To "Merge".
    - Edge Cases: AI misclassification, API errors, or input formatting issues.

  - **OpenRouter Chat Model**
    - Type: LangChain LLM Chat Model (OpenRouter)
    - Role: Provides the language model interface for "Is it a Business Problem?" node.
    - Configuration: Connected as AI model for classification.
    - Inputs: Receives prompts from "Is it a Business Problem?".
    - Outputs: Responses back to chain node.
    - Version Requirements: LangChain integration.
    - Edge Cases: API rate limits, authentication failures.

---

#### 1.3 Summarization

- **Overview:** Summarizes the filtered Reddit posts to concise content suitable for idea generation.
- **Nodes Involved:** 
  - Filter
  - Summarization Chain
  - OpenRouter Chat Model2
  - Merge1

- **Node Details:**

  - **Filter**
    - Type: Filter Node
    - Role: Filters out posts that do not meet certain criteria after merging.
    - Configuration: Presumably filters posts classified as business problems.
    - Inputs: From "Merge".
    - Outputs: To "Summarization Chain", "Merge1", and "Idea Generator".
    - Edge Cases: Overly restrictive filters may lose relevant data.

  - **Summarization Chain**
    - Type: LangChain Summarization Chain
    - Role: Uses AI to generate summaries of Reddit posts.
    - Configuration: Uses OpenRouter Chat Model2 as AI backend.
    - Inputs: From "Filter".
    - Outputs: To "Merge1".
    - Edge Cases: AI summarization errors, input size limits.

  - **OpenRouter Chat Model2**
    - Type: LangChain LLM Chat Model (OpenRouter)
    - Role: AI model instance used by the Summarization Chain.
    - Inputs: Receives summarization prompts.
    - Outputs: Summaries to "Summarization Chain".
    - Edge Cases: Connection or API failures.

  - **Merge1**
    - Type: Merge Node
    - Role: Combines summarized data and idea generation results.
    - Configuration: Always outputs data, ensuring downstream nodes receive input.
    - Inputs: From "Summarization Chain" and "Idea Generator".
    - Outputs: To "Code" node.
    - Edge Cases: Merge mismatches or missing data.

---

#### 1.4 Idea Generation

- **Overview:** Generates innovative business ideas from summarized Reddit posts using AI models.
- **Nodes Involved:** 
  - Idea Generator
  - OpenRouter Chat Model1

- **Node Details:**

  - **Idea Generator**
    - Type: LangChain LLM Chain
    - Role: Produces business ideas based on summarized input.
    - Configuration: Uses OpenRouter Chat Model1.
    - Inputs: From "Summarization Chain" (via OpenRouter Chat Model1).
    - Outputs: To "Merge1".
    - Edge Cases: AI model misinterpretation, API errors.

  - **OpenRouter Chat Model1**
    - Type: LangChain LLM Chat Model (OpenRouter)
    - Role: Backend AI model for "Idea Generator".
    - Inputs: Receives prompts for idea generation.
    - Outputs: Responses to "Idea Generator".
    - Edge Cases: API rate limits, authentication issues.

---

#### 1.5 Output & Storage

- **Overview:** Finalizes data output by processing with a code node and appending or updating rows in a Google Sheet.
- **Nodes Involved:** 
  - Code
  - Append or update row in sheet

- **Node Details:**

  - **Code**
    - Type: Code Node
    - Role: Executes custom JavaScript to transform or prepare data before storage.
    - Configuration: Custom script (not shown) tailored to structure data for Google Sheets.
    - Inputs: From "Merge1".
    - Outputs: To "Append or update row in sheet".
    - Edge Cases: Script errors, data format inconsistencies.

  - **Append or update row in sheet (Google Sheets)**
    - Type: Google Sheets Node
    - Role: Inserts or updates rows in a specified Google Sheet with generated business ideas.
    - Configuration: Requires Google OAuth2 credentials; sheet and range must be configured.
    - Inputs: From "Code" node.
    - Outputs: None (end node).
    - Edge Cases: Authentication failures, quota limits, incorrect sheet configuration.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                                | Input Node(s)                  | Output Node(s)                 | Sticky Note                                         |
|-----------------------------|----------------------------------|------------------------------------------------|-------------------------------|-------------------------------|----------------------------------------------------|
| When Executed by Another Workflow | Execute Workflow Trigger          | Entry point for external workflow execution     | None                          | Get many posts                |                                                    |
| Get many posts              | Reddit Node                      | Fetches multiple Reddit posts                    | When Executed by Another Workflow | If                            |                                                    |
| If                         | Conditional Node                 | Decides if posts meet criteria                    | Get many posts                | Edit Fields                  |                                                    |
| Edit Fields                | Set Node                        | Modifies/sets fields before AI processing         | If                            | Merge, Is it a Business Problem? |                                                    |
| Is it a Business Problem?  | LangChain LLM Chain             | AI classification if post describes a business problem | Edit Fields                  | Merge                         |                                                    |
| OpenRouter Chat Model      | LangChain LLM Chat Model        | AI model for business problem classification     | Is it a Business Problem?     | Is it a Business Problem?     |                                                    |
| Merge                      | Merge Node                      | Combines edited posts and classification results | Edit Fields, Is it a Business Problem? | Filter                       |                                                    |
| Filter                     | Filter Node                    | Filters posts classified as business problems    | Merge                        | Summarization Chain, Merge1, Idea Generator |                                                    |
| OpenRouter Chat Model2     | LangChain LLM Chat Model        | AI model for summarization                         | Summarization Chain           | Summarization Chain           |                                                    |
| Summarization Chain        | LangChain Summarization Chain  | Summarizes filtered posts                          | Filter                       | Merge1                       |                                                    |
| Idea Generator             | LangChain LLM Chain             | Generates business ideas from summaries           | Filter                       | Merge1                       |                                                    |
| OpenRouter Chat Model1     | LangChain LLM Chat Model        | AI model for idea generation                       | Idea Generator               | Idea Generator               |                                                    |
| Merge1                     | Merge Node                      | Combines summaries and generated ideas            | Summarization Chain, Idea Generator | Code                        |                                                    |
| Code                       | Code Node                      | Processes final data before storage                | Merge1                       | Append or update row in sheet |                                                    |
| Append or update row in sheet | Google Sheets Node             | Stores final ideas in Google Sheets                | Code                        | None                         |                                                    |
| Sticky Note1               | Sticky Note                    | (Empty content)                                    | None                        | None                         |                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Node:**
   - Add **Execute Workflow Trigger** node named "When Executed by Another Workflow".
   - Use default settings.

2. **Fetch Reddit Posts:**
   - Add **Reddit** node named "Get many posts".
   - Configure subreddit(s), search terms, or filters as needed.
   - Connect "When Executed by Another Workflow" → "Get many posts".

3. **Conditional Check on Posts:**
   - Add **If** node named "If".
   - Configure condition to check if posts array is non-empty or meets criteria.
   - Connect "Get many posts" → "If" (main output true).

4. **Edit/Set Fields:**
   - Add **Set** node named "Edit Fields".
   - Configure to add or modify fields required for AI chains (e.g., extract post text).
   - Connect "If" → "Edit Fields".

5. **Business Problem Classification:**
   - Add **LangChain LLM Chain** node named "Is it a Business Problem?".
   - Configure prompt to classify if input text describes a business problem.
   - Add **LangChain LLM Chat Model (OpenRouter)** node named "OpenRouter Chat Model".
   - Connect "Edit Fields" → "OpenRouter Chat Model" (ai_languageModel input).
   - Connect "OpenRouter Chat Model" → "Is it a Business Problem?" (ai_languageModel output).

6. **Merge Edited Fields and Classification:**
   - Add **Merge** node named "Merge".
   - Connect "Edit Fields" and "Is it a Business Problem?" → "Merge".

7. **Filter Business Problem Posts:**
   - Add **Filter** node named "Filter".
   - Configure to pass only posts classified as business problems.
   - Connect "Merge" → "Filter".

8. **Summarization:**
   - Add **LangChain Summarization Chain** node named "Summarization Chain".
   - Add **LangChain LLM Chat Model (OpenRouter)** node named "OpenRouter Chat Model2".
   - Connect "Filter" → "Summarization Chain".
   - Connect "OpenRouter Chat Model2" → "Summarization Chain" (ai_languageModel).

9. **Idea Generation:**
   - Add **LangChain LLM Chain** node named "Idea Generator".
   - Add **LangChain LLM Chat Model (OpenRouter)** node named "OpenRouter Chat Model1".
   - Connect "Filter" → "Idea Generator".
   - Connect "OpenRouter Chat Model1" → "Idea Generator" (ai_languageModel).

10. **Merge Summaries and Ideas:**
    - Add **Merge** node named "Merge1".
    - Configure to always output data.
    - Connect "Summarization Chain" → "Merge1".
    - Connect "Idea Generator" → "Merge1".

11. **Data Processing with Code Node:**
    - Add **Code** node named "Code".
    - Write JavaScript to transform merged data as needed for Google Sheets.
    - Connect "Merge1" → "Code".

12. **Google Sheets Storage:**
    - Add **Google Sheets** node named "Append or update row in sheet".
    - Configure with OAuth2 credentials and target spreadsheet/range.
    - Connect "Code" → "Append or update row in sheet".

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                             |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------|
| This workflow uses OpenRouter models for AI processing, enabling flexible language model integration. | Documentation: https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/ |
| Make sure to configure Reddit API credentials properly to avoid rate limits and authentication errors. | Reddit API docs: https://www.reddit.com/dev/api/            |
| Google Sheets node requires OAuth2 credentials with write access to the targeted spreadsheet.       | Google Sheets API: https://developers.google.com/sheets/api  |

---

**Disclaimer:** The provided text is sourced exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.