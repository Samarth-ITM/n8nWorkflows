Extract Business Leads from Reddit using GPT-4.1-mini Analysis and Google Sheets

https://n8nworkflows.xyz/workflows/extract-business-leads-from-reddit-using-gpt-4-1-mini-analysis-and-google-sheets-3849


# Extract Business Leads from Reddit using GPT-4.1-mini Analysis and Google Sheets

### 1. Workflow Overview

This n8n workflow automates the extraction and analysis of business leads from Reddit posts within a specific subreddit. It searches Reddit for posts matching given keywords, filters posts based on engagement and recency, uses OpenRouter’s GPT-4.1-mini AI model to determine business relevance and generate summaries, and finally logs the curated leads into a Google Sheets document for tracking and further use.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Manual trigger initiates the workflow.
- **1.2 Data Extraction:** Reddit node fetches posts from a targeted subreddit based on keywords and sorting criteria.
- **1.3 Filtering:** Posts are filtered by engagement metrics (upvotes), content presence, and recency.
- **1.4 AI Processing:** Two-step AI analysis using OpenRouter GPT-4.1-mini models — first to classify posts as business opportunities, then to generate executive summaries.
- **1.5 Data Transformation:** Selection and formatting of key fields from Reddit posts and AI outputs.
- **1.6 Output:** Appending the processed and summarized data into a Google Sheets spreadsheet.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Starts the workflow manually when triggered by the user.
- **Nodes Involved:**  
  - When clicking ‘Test workflow’

- **Node Details:**  
  - **Type:** Manual Trigger  
  - **Role:** Entry point for manual execution of the workflow.  
  - **Configuration:** No parameters; simply triggers the workflow on user action.  
  - **Connections:** Outputs to "Get Posts" node.  
  - **Edge Cases:** None significant; failure only if manual trigger is not activated.

#### 2.2 Data Extraction

- **Overview:** Retrieves recent Reddit posts from a specific subreddit filtered by keyword and sorted by popularity.
- **Nodes Involved:**  
  - Get Posts  
  - Filter Posts By Features  
  - Select Key Fields

- **Node Details:**

  - **Get Posts**  
    - **Type:** Reddit  
    - **Role:** Searches the "Entrepreneur" subreddit for posts containing the keyword "how do I find leads" sorted by "hot".  
    - **Configuration:**  
      - Operation: Search  
      - Subreddit: Entrepreneur  
      - Keyword: "how do I find leads"  
      - Sort: hot  
    - **Connections:** Outputs to "Filter Posts By Features".  
    - **Edge Cases:**  
      - OAuth 2.0 authentication failure if Reddit credentials are invalid or expired.  
      - API rate limits or downtime.  
      - No posts found for the keyword.

  - **Filter Posts By Features**  
    - **Type:** If (Conditional)  
    - **Role:** Filters posts to keep only those with more than 2 upvotes, non-empty content, and posted within the last 180 days.  
    - **Configuration:**  
      - Conditions combined with AND:  
        - ups > 2  
        - selftext not empty  
        - created date after (today - 180 days)  
    - **Connections:** Passes filtered posts to "Select Key Fields".  
    - **Edge Cases:**  
      - Posts missing expected fields (e.g., ups or selftext) may cause expression errors.  
      - Timezone considerations in date filtering.

  - **Select Key Fields**  
    - **Type:** Set  
    - **Role:** Extracts and renames key fields from Reddit posts for downstream processing.  
    - **Configuration:**  
      - Assigns:  
        - upvotes = ups  
        - subreddit_subscribers  
        - postcontent = selftext  
        - url  
        - date = converted created timestamp to ISO string  
    - **Connections:** Outputs to both "Merge Input" and "Basic LLM Chain".  
    - **Edge Cases:**  
      - Missing or malformed data fields could cause errors in date conversion or assignment.

#### 2.3 Filtering by Business Relevance

- **Overview:** Uses AI to classify posts as relevant business opportunities or not.
- **Nodes Involved:**  
  - Basic LLM Chain  
  - OpenRouter Chat Model  
  - Merge Input  
  - Filter Posts By Content

- **Node Details:**

  - **OpenRouter Chat Model**  
    - **Type:** LangChain OpenRouter Chat Model  
    - **Role:** Provides GPT-4.1-mini AI model for classification.  
    - **Configuration:**  
      - Model: openai/gpt-4.1-mini  
      - No additional options specified.  
    - **Connections:** Outputs to "Basic LLM Chain".  
    - **Edge Cases:**  
      - API key invalid or quota exceeded.  
      - Timeout or network errors.  
      - Unexpected AI output format.

  - **Basic LLM Chain**  
    - **Type:** LangChain LLM Chain  
    - **Role:** Sends prompt to AI to decide if post describes a business problem or need (expects "yes" or "no").  
    - **Configuration:**  
      - Prompt: "Decide whether this reddit post is describing a business-related problem or a need for a solution. Output only yes or no."  
      - Input variable: postcontent from "Select Key Fields".  
    - **Connections:** Outputs to "Merge Input".  
    - **Edge Cases:**  
      - AI returns unexpected answers other than "yes" or "no".  
      - Expression failure if postcontent missing.

  - **Merge Input**  
    - **Type:** Merge  
    - **Role:** Combines outputs from "Select Key Fields" and "Basic LLM Chain" by position to unify Reddit data with AI classification.  
    - **Configuration:**  
      - Mode: Combine  
      - Combine by position  
    - **Connections:** Outputs to "Filter Posts By Content".  
    - **Edge Cases:**  
      - Mismatched input array lengths cause missing data.

  - **Filter Posts By Content**  
    - **Type:** If (Conditional)  
    - **Role:** Filters posts where AI output classification equals "yes" (business opportunity).  
    - **Configuration:**  
      - Condition: output == "yes"  
    - **Connections:** Outputs to "Post Summarization" and then "Edit Fields1".  
    - **Edge Cases:**  
      - Case sensitivity issues if AI outputs "Yes" or "YES".  
      - Missing output field.

#### 2.4 AI Summarization and Insight Generation

- **Overview:** Generates executive summaries of filtered posts using AI and prepares data for output.
- **Nodes Involved:**  
  - Post Summarization  
  - OpenRouter Chat Model1  
  - Edit Fields  
  - Edit Fields1  
  - Merge 3 Inputs

- **Node Details:**

  - **OpenRouter Chat Model1**  
    - **Type:** LangChain OpenRouter Chat Model  
    - **Role:** Provides GPT-4.1-mini AI model for summarization.  
    - **Configuration:**  
      - Model: openai/gpt-4.1-mini  
    - **Connections:** Outputs to "Post Summarization".  
    - **Edge Cases:**  
      - Same as previous OpenRouter node.

  - **Post Summarization**  
    - **Type:** LangChain Chain Summarization  
    - **Role:** Summarizes the content of the Reddit post into an executive summary.  
    - **Configuration:**  
      - Default summarization options.  
    - **Connections:** Outputs to "Edit Fields".  
    - **Edge Cases:**  
      - AI summarization failure or empty summary.

  - **Edit Fields**  
    - **Type:** Set  
    - **Role:** Adds a "summary" field containing the AI-generated summary text.  
    - **Configuration:**  
      - Assigns summary = response.text from AI output.  
    - **Connections:** Outputs to "Merge 3 Inputs".  
    - **Edge Cases:**  
      - Missing or malformed AI response.

  - **Edit Fields1**  
    - **Type:** Set  
    - **Role:** Prepares final data fields including date, subreddit subscribers, URL, upvotes, post content, and business opportunity flag for output.  
    - **Configuration:**  
      - Assigns multiple fields from merged data, including "business_opportunity" from AI classification.  
    - **Connections:** Outputs to "Merge 3 Inputs".  
    - **Edge Cases:**  
      - Missing fields from previous merges.

  - **Merge 3 Inputs**  
    - **Type:** Merge  
    - **Role:** Combines the summarized data ("Edit Fields") and the detailed data ("Edit Fields1") into one dataset for output.  
    - **Configuration:**  
      - Mode: Combine  
      - Combine by position  
    - **Connections:** Outputs to "Output The Results".  
    - **Edge Cases:**  
      - Mismatched array lengths.

#### 2.5 Output to Google Sheets

- **Overview:** Appends the processed and summarized lead data into a Google Sheets spreadsheet.
- **Nodes Involved:**  
  - Output The Results

- **Node Details:**

  - **Output The Results**  
    - **Type:** Google Sheets  
    - **Role:** Appends rows to a specified sheet within a Google Sheets document.  
    - **Configuration:**  
      - Operation: Append  
      - Document ID: "1cIMIh_DjoWXMDaJEH-AyTZbnAha6TxthCSSEam4NLsE"  
      - Sheet Name: "Find-Leads" (sheet ID 979106892)  
      - Columns: Auto-map input data fields to sheet columns  
    - **Connections:** Terminal node.  
    - **Edge Cases:**  
      - OAuth 2.0 authentication failure if Google credentials invalid or expired.  
      - API quota exceeded or rate limits.  
      - Sheet not found or access denied.  
      - Data type mismatches.

---

### 3. Summary Table

| Node Name                 | Node Type                         | Functional Role                                  | Input Node(s)                    | Output Node(s)                  | Sticky Note                                                                                                  |
|---------------------------|----------------------------------|-------------------------------------------------|---------------------------------|--------------------------------|--------------------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                   | Entry point to start workflow manually           | -                               | Get Posts                      |                                                                                                              |
| Get Posts                 | Reddit                           | Fetch Reddit posts by keyword and subreddit      | When clicking ‘Test workflow’    | Filter Posts By Features        | # Data Extraction<br>## Retrieves recent posts from specific Reddit community (subreddit)<br>## Filters content by keywords and upvotes |
| Filter Posts By Features  | If                              | Filter posts by upvotes, content presence, recency | Get Posts                      | Select Key Fields               |                                                                                                              |
| Select Key Fields         | Set                             | Extract and rename key Reddit post fields        | Filter Posts By Features         | Merge Input, Basic LLM Chain    |                                                                                                              |
| Basic LLM Chain           | LangChain LLM Chain             | Classify posts as business-related or not        | OpenRouter Chat Model            | Merge Input                    | # Transformation Step<br>## Analyze using LLM (AI)<br>## Filter for business opportunities                    |
| OpenRouter Chat Model     | LangChain OpenRouter Chat Model | Provide GPT-4.1-mini AI model for classification | Basic LLM Chain                 | Basic LLM Chain                |                                                                                                              |
| Merge Input               | Merge                           | Combine Reddit data and AI classification         | Select Key Fields, Basic LLM Chain | Filter Posts By Content         |                                                                                                              |
| Filter Posts By Content   | If                              | Filter posts where AI output is "yes"             | Merge Input                    | Post Summarization, Edit Fields1 |                                                                                                              |
| Post Summarization        | LangChain Chain Summarization   | Generate executive summary of post content        | OpenRouter Chat Model1           | Edit Fields                   | #Transformation 2nd Step + Load to Gsheet<br>## Insight Generation<br>## Generates executive summaries of key opportunities<br>## Submit findings in Google Sheets |
| OpenRouter Chat Model1    | LangChain OpenRouter Chat Model | Provide GPT-4.1-mini AI model for summarization  | Post Summarization              | Post Summarization            |                                                                                                              |
| Edit Fields               | Set                             | Add summary field from AI output                   | Post Summarization              | Merge 3 Inputs                |                                                                                                              |
| Edit Fields1              | Set                             | Prepare final data fields for output               | Filter Posts By Content         | Merge 3 Inputs                |                                                                                                              |
| Merge 3 Inputs            | Merge                           | Combine summarized and detailed data               | Edit Fields, Edit Fields1       | Output The Results            |                                                                                                              |
| Output The Results        | Google Sheets                   | Append processed data to Google Sheets             | Merge 3 Inputs                 | -                            |                                                                                                              |
| Sticky Note               | Sticky Note                     | Comment on Data Extraction block                    | -                             | -                            | # Data Extraction<br>## Retrieves recent posts from specific Reddit community (subreddit)<br>## Filters content by keywords and upvotes |
| Sticky Note2              | Sticky Note                     | Comment on Transformation Step                      | -                             | -                            | # Transformation Step<br>## Analyze using LLM (AI)<br>## Filter for business opportunities                    |
| Sticky Note3              | Sticky Note                     | Comment on Summarization and Output                 | -                             | -                            | #Transformation 2nd Step + Load to Gsheet<br>## Insight Generation<br>## Generates executive summaries of key opportunities<br>## Submit findings in Google Sheets |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Create Reddit Node ("Get Posts")**  
   - Type: Reddit  
   - Operation: Search  
   - Subreddit: Entrepreneur  
   - Keyword: "how do I find leads"  
   - Sort: hot  
   - Credentials: Select Reddit OAuth2 credentials (requires Reddit app with Client ID & Secret).  
   - Connect Manual Trigger output to this node.

3. **Create If Node ("Filter Posts By Features")**  
   - Type: If  
   - Conditions (AND):  
     - Numeric: ups > 2  
     - String: selftext not empty  
     - DateTime: created date after (today minus 180 days)  
   - Connect "Get Posts" output to this node.

4. **Create Set Node ("Select Key Fields")**  
   - Type: Set  
   - Assign fields:  
     - upvotes = ups (string)  
     - subreddit_subscribers = subreddit_subscribers (number)  
     - postcontent = selftext (string)  
     - url = url (string)  
     - date = convert created (epoch seconds) to ISO string (using expression)  
   - Connect "Filter Posts By Features" true output to this node.

5. **Create OpenRouter Chat Model Node ("OpenRouter Chat Model")**  
   - Type: LangChain OpenRouter Chat Model  
   - Model: openai/gpt-4.1-mini  
   - Credentials: Select OpenRouter API key.  
   - Connect output to "Basic LLM Chain".

6. **Create LangChain LLM Chain Node ("Basic LLM Chain")**  
   - Type: LangChain LLM Chain  
   - Prompt:  
     - Human message: "Reddit post: {{ $json.postcontent }}"  
     - System message: "The post should mention a specific challenge or requirement that a business is trying to address. Is this post about a business problem or need for a solution? Output only yes or no."  
   - Connect "OpenRouter Chat Model" output to this node.

7. **Create Merge Node ("Merge Input")**  
   - Type: Merge  
   - Mode: Combine  
   - Combine by: Position  
   - Connect outputs of "Select Key Fields" and "Basic LLM Chain" to this node.

8. **Create If Node ("Filter Posts By Content")**  
   - Type: If  
   - Condition: output == "yes" (case sensitive)  
   - Connect "Merge Input" output to this node.

9. **Create OpenRouter Chat Model Node ("OpenRouter Chat Model1")**  
   - Type: LangChain OpenRouter Chat Model  
   - Model: openai/gpt-4.1-mini  
   - Credentials: OpenRouter API key (same as before).  
   - Connect output to "Post Summarization".

10. **Create LangChain Chain Summarization Node ("Post Summarization")**  
    - Type: LangChain Chain Summarization  
    - Use default summarization options.  
    - Connect "OpenRouter Chat Model1" output to this node.

11. **Create Set Node ("Edit Fields")**  
    - Type: Set  
    - Assign: summary = {{ $json.response.text }} (AI summary text)  
    - Connect "Post Summarization" output to this node.

12. **Create Set Node ("Edit Fields1")**  
    - Type: Set  
    - Assign fields from filtered merged data:  
      - date  
      - subreddit_subscribers  
      - url  
      - upvotes  
      - postcontent  
      - business_opportunity = output (AI classification)  
    - Connect "Filter Posts By Content" true output to this node.

13. **Create Merge Node ("Merge 3 Inputs")**  
    - Type: Merge  
    - Mode: Combine  
    - Combine by: Position  
    - Connect outputs of "Edit Fields" and "Edit Fields1" to this node.

14. **Create Google Sheets Node ("Output The Results")**  
    - Type: Google Sheets  
    - Operation: Append  
    - Document ID: "1cIMIh_DjoWXMDaJEH-AyTZbnAha6TxthCSSEam4NLsE"  
    - Sheet Name: "Find-Leads" (ID 979106892)  
    - Columns: Auto-map input data fields  
    - Credentials: Google Sheets OAuth2 credentials (requires Google Cloud Console setup with Sheets API enabled and OAuth Client ID).  
    - Connect "Merge 3 Inputs" output to this node.

15. **Connect all nodes as per above connections** to ensure data flows from manual trigger through Reddit search, filtering, AI classification, summarization, and finally to Google Sheets.

16. **Test the workflow** by manually triggering and verifying data appears correctly in Google Sheets.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Watch the full setup tutorial on how to build this ETL pipeline using n8n: https://youtu.be/F3-fbU3UmYQ | Workflow overview and detailed setup video tutorial                                             |
| Reddit OAuth 2.0 credential setup requires creating a Reddit app: https://youtu.be/zlGXtW4LAK8      | For Reddit API authentication                                                                   |
| OpenRouter API key generation tutorial: https://youtu.be/Cq5Y3zpEhlc                                 | For AI model authentication                                                                     |
| Google Sheets OAuth 2.0 setup requires enabling Sheets API and creating OAuth Client ID in Google Cloud Console | For Google Sheets node authentication                                                           |
| Sticky notes in workflow clarify blocks: Data Extraction, Transformation, and Output steps          | Provides conceptual grouping and explanation within the workflow editor                          |

---

This documentation provides a comprehensive understanding of the workflow’s structure, logic, and configuration to enable reproduction, modification, and troubleshooting by advanced users or AI agents.