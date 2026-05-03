Discover Business Ideas from HackerNews Posts with GPT-4.1 Analysis and Google Sheets

https://n8nworkflows.xyz/workflows/discover-business-ideas-from-hackernews-posts-with-gpt-4-1-analysis-and-google-sheets-3877


# Discover Business Ideas from HackerNews Posts with GPT-4.1 Analysis and Google Sheets

### 1. Workflow Overview

This workflow is designed to discover business ideas by analyzing top posts on Hacker News using GPT-4.1 AI and storing the results in Google Sheets. It targets users interested in identifying emerging pain points and opportunities from popular tech news discussions.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger to start the workflow and fetch Hacker News posts.
- **1.2 Filtering and Selection:** Filter posts based on engagement metrics and select key metadata fields.
- **1.3 Content Retrieval and AI Analysis:** Fetch article content, run GPT-4.1 AI agent to detect and summarize business pain points.
- **1.4 Output Preparation and Storage:** Format AI results and append them to a Google Sheet for further analysis.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block initiates the workflow manually and retrieves the latest top posts from Hacker News.

**Nodes Involved:**  
- When clicking ‘Test workflow’  
- Hacker News

**Node Details:**

- **When clicking ‘Test workflow’**  
  - Type: Manual Trigger  
  - Role: Starts the workflow on user command.  
  - Configuration: Default manual trigger with no parameters.  
  - Inputs: None  
  - Outputs: Hacker News node  
  - Edge Cases: None typical; user must manually trigger.  

- **Hacker News**  
  - Type: Hacker News node  
  - Role: Fetches current top posts from Hacker News front page.  
  - Configuration: Default settings to retrieve front page posts.  
  - Inputs: Manual Trigger node  
  - Outputs: Filter Posts By Features node  
  - Edge Cases: Possible API downtime or rate limits; no authentication required.  

---

#### 2.2 Filtering and Selection

**Overview:**  
Filters Hacker News posts by engagement criteria (comments > 80, points > 200, recent date) and extracts key metadata fields for further processing.

**Nodes Involved:**  
- Filter Posts By Features  
- Select Key Fields

**Node Details:**

- **Filter Posts By Features**  
  - Type: If node  
  - Role: Applies conditional filters on posts based on comments count, points, and date.  
  - Configuration: Conditions set to check if comments > 80 AND points > 200 AND post date is recent.  
  - Inputs: Hacker News node  
  - Outputs: Select Key Fields node  
  - Edge Cases: Posts with missing data fields may cause expression errors; ensure null checks.  

- **Select Key Fields**  
  - Type: Set node  
  - Role: Extracts and formats essential metadata fields such as title, URL, points, comments, and date.  
  - Configuration: Fields explicitly set to be passed downstream.  
  - Inputs: Filter Posts By Features node  
  - Outputs: Merge Input and Loop Over Items nodes  
  - Edge Cases: Missing or malformed fields could propagate errors; validate field existence.  

---

#### 2.3 Content Retrieval and AI Analysis

**Overview:**  
Fetches article content via HTTP requests, runs a GPT-4.1-based AI agent to analyze and summarize business pain points, and parses structured AI output.

**Nodes Involved:**  
- Loop Over Items  
- HTTP Request  
- AI Agent  
- OpenRouter Chat Model  
- Structured Output Parser  
- If Topic1

**Node Details:**

- **Loop Over Items**  
  - Type: SplitInBatches  
  - Role: Processes posts one by one or in batches to manage API calls and processing load.  
  - Configuration: Default batch size or customized to control throughput.  
  - Inputs: Select Key Fields node (also connected from If Topic1 for loop continuation)  
  - Outputs: HTTP Request (ai_tool) and AI Agent (main)  
  - Edge Cases: Large batch sizes may cause timeouts; empty batches should be handled gracefully.  

- **HTTP Request**  
  - Type: HTTP Request Tool  
  - Role: Fetches full article content from the post URL for AI analysis.  
  - Configuration: Configured to GET the article URL extracted from post metadata.  
  - Inputs: Loop Over Items node (ai_tool)  
  - Outputs: AI Agent node (ai_tool)  
  - Edge Cases: HTTP errors (404, 500), timeouts, redirects, or content not accessible; must handle gracefully.  

- **AI Agent**  
  - Type: Langchain Agent  
  - Role: Runs GPT-4.1 AI analysis on fetched content to detect business pain points.  
  - Configuration: Uses OpenRouter Chat Model and Structured Output Parser as subcomponents.  
  - Inputs: HTTP Request (ai_tool), Loop Over Items (main), OpenRouter Chat Model (ai_languageModel), Structured Output Parser (ai_outputParser)  
  - Outputs: If Topic1 node  
  - Edge Cases: AI model errors, rate limits, malformed input data, or parsing failures.  

- **OpenRouter Chat Model**  
  - Type: Langchain OpenRouter Chat Model  
  - Role: Provides GPT-4.1 language model interface for AI Agent.  
  - Configuration: Uses API key credential from OpenRouter account.  
  - Inputs: AI Agent (ai_languageModel)  
  - Outputs: AI Agent (ai_languageModel)  
  - Edge Cases: Authentication errors, API key invalid or expired, rate limits.  

- **Structured Output Parser**  
  - Type: Langchain Structured Output Parser  
  - Role: Parses AI Agent’s output into structured data for downstream processing.  
  - Configuration: Predefined schema for expected AI output format.  
  - Inputs: AI Agent (ai_outputParser)  
  - Outputs: AI Agent (ai_outputParser)  
  - Edge Cases: Parsing errors if AI output deviates from schema.  

- **If Topic1**  
  - Type: If node  
  - Role: Checks if AI analysis detected relevant business pain points (topic presence).  
  - Configuration: Condition based on AI Agent output fields indicating topic relevance.  
  - Inputs: AI Agent node  
  - Outputs: Assign Sheet Headers (if true), Loop Over Items (if false)  
  - Edge Cases: False negatives or positives due to AI output ambiguity.  

---

#### 2.4 Output Preparation and Storage

**Overview:**  
Prepares the final data structure with headers and appends the results to a Google Sheet for further analysis.

**Nodes Involved:**  
- Assign Sheet Headers  
- Merge Input  
- Output The Results

**Node Details:**

- **Assign Sheet Headers**  
  - Type: Set node  
  - Role: Adds or confirms column headers for Google Sheets output.  
  - Configuration: Sets fields such as Title, URL, Points, Comments, Date, and AI Summary.  
  - Inputs: If Topic1 node (true branch)  
  - Outputs: Merge Input node  
  - Edge Cases: Missing fields could cause misaligned columns in Sheets.  

- **Merge Input**  
  - Type: Merge node  
  - Role: Combines data streams for final output.  
  - Configuration: Default merge mode to consolidate data before output.  
  - Inputs: Select Key Fields node and Assign Sheet Headers node  
  - Outputs: Output The Results node  
  - Edge Cases: Data mismatch or empty inputs could cause errors.  

- **Output The Results**  
  - Type: Google Sheets node  
  - Role: Appends the processed data rows to a configured Google Sheet.  
  - Configuration: Uses OAuth2 credentials; target spreadsheet and sheet specified.  
  - Inputs: Merge Input node  
  - Outputs: None (end of workflow)  
  - Edge Cases: Authentication failures, quota limits, sheet access permissions, or malformed data rows.  

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                          | Input Node(s)               | Output Node(s)               | Sticky Note                     |
|-------------------------|----------------------------------|----------------------------------------|----------------------------|-----------------------------|--------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Starts the workflow manually            | None                       | Hacker News                 |                                |
| Hacker News             | Hacker News node                  | Fetch top posts from Hacker News        | When clicking ‘Test workflow’ | Filter Posts By Features    |                                |
| Filter Posts By Features | If node                          | Filter posts by comments, points, date | Hacker News                | Select Key Fields           |                                |
| Select Key Fields       | Set node                         | Extract key metadata fields              | Filter Posts By Features    | Merge Input, Loop Over Items |                                |
| Loop Over Items         | SplitInBatches                   | Process posts in batches                 | Select Key Fields, If Topic1 | HTTP Request (ai_tool), AI Agent (main) |                                |
| HTTP Request            | HTTP Request Tool                | Fetch article content                    | Loop Over Items (ai_tool)   | AI Agent (ai_tool)          |                                |
| AI Agent                | Langchain Agent                  | Run GPT-4.1 AI analysis                  | HTTP Request (ai_tool), Loop Over Items (main), OpenRouter Chat Model (ai_languageModel), Structured Output Parser (ai_outputParser) | If Topic1                   |                                |
| OpenRouter Chat Model   | Langchain OpenRouter Chat Model | Provide GPT-4.1 language model          | AI Agent (ai_languageModel) | AI Agent (ai_languageModel) | Requires OpenRouter API key credential |
| Structured Output Parser| Langchain Structured Output Parser | Parse AI output into structured data    | AI Agent (ai_outputParser)  | AI Agent (ai_outputParser)  |                                |
| If Topic1               | If node                         | Check if AI detected relevant topics    | AI Agent                   | Assign Sheet Headers (true), Loop Over Items (false) |                                |
| Assign Sheet Headers    | Set node                        | Add headers for Google Sheets output    | If Topic1                  | Merge Input                 |                                |
| Merge Input             | Merge node                     | Combine data streams for output         | Select Key Fields, Assign Sheet Headers | Output The Results          |                                |
| Output The Results      | Google Sheets node             | Append data to Google Sheet              | Merge Input                | None                       | Requires Google Sheets OAuth2 credential |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node:**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add Hacker News Node:**  
   - Type: Hacker News  
   - Configuration: Default to fetch front page posts.  
   - Connect Manual Trigger → Hacker News.

3. **Add If Node for Filtering Posts:**  
   - Name: Filter Posts By Features  
   - Conditions:  
     - Comments count > 80  
     - Points > 200  
     - Post date is recent (e.g., within last few days)  
   - Connect Hacker News → Filter Posts By Features.

4. **Add Set Node to Select Key Fields:**  
   - Name: Select Key Fields  
   - Fields to set: title, URL, points, comments, date, etc.  
   - Connect Filter Posts By Features → Select Key Fields.

5. **Add SplitInBatches Node:**  
   - Name: Loop Over Items  
   - Configure batch size (e.g., 1 or small number to avoid API overload).  
   - Connect Select Key Fields → Loop Over Items.

6. **Add HTTP Request Node:**  
   - Name: HTTP Request  
   - Configure to GET the article URL from current batch item.  
   - Connect Loop Over Items (ai_tool output) → HTTP Request.

7. **Add Langchain OpenRouter Chat Model Node:**  
   - Name: OpenRouter Chat Model  
   - Configure with OpenRouter API Key credential.  
   - Connect to AI Agent node (ai_languageModel input).

8. **Add Langchain Structured Output Parser Node:**  
   - Name: Structured Output Parser  
   - Configure expected output schema for AI results.  
   - Connect to AI Agent node (ai_outputParser input).

9. **Add Langchain Agent Node:**  
   - Name: AI Agent  
   - Configure to use OpenRouter Chat Model and Structured Output Parser.  
   - Connect HTTP Request (ai_tool) → AI Agent (ai_tool input).  
   - Connect Loop Over Items (main) → AI Agent (main input).  
   - Connect OpenRouter Chat Model → AI Agent (ai_languageModel).  
   - Connect Structured Output Parser → AI Agent (ai_outputParser).

10. **Add If Node to Check AI Output:**  
    - Name: If Topic1  
    - Condition: Check if AI output indicates relevant business pain points.  
    - Connect AI Agent → If Topic1.

11. **Add Set Node to Assign Sheet Headers:**  
    - Name: Assign Sheet Headers  
    - Set headers: Title, URL, Points, Comments, Date, AI Summary, etc.  
    - Connect If Topic1 (true branch) → Assign Sheet Headers.

12. **Add Merge Node:**  
    - Name: Merge Input  
    - Connect Select Key Fields → Merge Input (input 1).  
    - Connect Assign Sheet Headers → Merge Input (input 2).

13. **Add Google Sheets Node:**  
    - Name: Output The Results  
    - Configure OAuth2 credentials for Google Sheets.  
    - Specify target spreadsheet and sheet name.  
    - Connect Merge Input → Output The Results.

14. **Connect If Topic1 (false branch) → Loop Over Items:**  
    - To continue processing next batch if no relevant topic found.

15. **Test the workflow:**  
    - Trigger manually and verify data flows correctly.  
    - Monitor for errors such as API limits or missing data.

---

### 5. General Notes & Resources

| Note Content                                                                                         | Context or Link                                                                                   |
|----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| OpenRouter API Key is required for GPT-4.1 model usage. Generate from your OpenRouter account.     | https://youtu.be/Cq5Y3zpEhlc (YouTube tutorial)                                                 |
| Google Sheets OAuth2 setup requires enabling Sheets API and creating OAuth Client ID in Google Cloud Console. | n8n documentation and Google Cloud Console setup instructions                                   |
| Workflow filters posts with comments > 80 and points > 200 to focus on highly engaged Hacker News posts. | Filtering criteria ensures quality input for AI analysis                                        |
| AI Agent uses Langchain integration with OpenRouter and structured output parsing for reliable data extraction. | Ensures AI output is consistent and machine-readable                                            |
| Batch processing via SplitInBatches node prevents API overload and manages rate limits.             | Important for scalability and error handling                                                    |

---

This document provides a detailed, structured reference for understanding, reproducing, and maintaining the "Discover Business Ideas from HackerNews Posts with GPT-4.1 Analysis and Google Sheets" workflow.