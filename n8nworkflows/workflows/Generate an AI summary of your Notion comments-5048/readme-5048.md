Generate an AI summary of your Notion comments

https://n8nworkflows.xyz/workflows/generate-an-ai-summary-of-your-notion-comments-5048


# Generate an AI summary of your Notion comments

### 1. Workflow Overview

This workflow automates the process of generating AI-based summaries of comments on Notion pages within a specified database. It is designed to help users quickly understand conversations or comment threads on their Notion pages by summarizing them into concise textual summaries using an AI language model.

The workflow is logically divided into the following functional blocks:

- **1.1 Trigger and Initialization:** Schedules the workflow execution and sets key variables related to the Notion database and AI summary properties.
- **1.2 Data Retrieval:** Queries the Notion database to list pages and fetches comments associated with each page.
- **1.3 Filtering:** Filters out pages that have not received new comments since the last workflow execution to optimize processing.
- **1.4 AI Summarization:** Uses an AI language model (Google Gemini) to summarize the comments of each relevant Notion page.
- **1.5 Updating Notion:** Updates the Notion page with the generated AI summary and records the timestamp of the last execution.

Supporting the blocks, several sticky notes provide instructions, tips, and notes for adaptation and credential setup.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Initialization

- **Overview:**  
  This block triggers the workflow periodically and initializes necessary variables such as the Notion database ID and property names for storing AI summaries and execution timestamps.

- **Nodes Involved:**  
  - Run every hour  
  - Define your Notion Database  
  - Sticky Note5  
  - Sticky Note6  
  - Sticky Note7

- **Node Details:**

  - **Run every hour**  
    - *Type:* Schedule Trigger  
    - *Role:* Triggers the workflow every hour to check for new comments.  
    - *Configuration:* Interval set to every 1 hour.  
    - *Connections:* Outputs to "Define your Notion Database" node.  
    - *Edge Cases:* If the workflow runs while Notion API is down or rate-limited, potential errors could occur.  
    - *Notes:* Using webhooks is recommended for efficiency (see Sticky Note).

  - **Define your Notion Database**  
    - *Type:* Set Node  
    - *Role:* Defines key variables used throughout the workflow:  
      - `notionDatabaseId`: ID of the Notion database to monitor  
      - `AI Summary Notion Property (type text)`: Property name where AI summary will be stored  
      - `Last Execution Notion Property (type date)`: Property name for last execution date  
    - *Configuration:* Values are hardcoded strings, e.g., database ID `"20d45c70c57381f09418d42c78ad360b"`.  
    - *Connections:* Outputs to "List database pages".  
    - *Edge Cases:* Incorrect or expired database ID or property names will cause errors downstream.

  - **Sticky Note5, Sticky Note6, Sticky Note7**  
    - *Type:* Sticky Note  
    - *Role:* Provide instructions about defining variables and setting Notion credentials.  
    - *Content Highlights:*  
      - Step 1: Define variables (database ID, property names)  
      - Step 2 & 3: Set Notion credentials via internal integration at https://www.notion.so/profile/integrations  
    - *Connections:* None (informational only).

#### 2.2 Data Retrieval

- **Overview:**  
  This block fetches all pages from the specified Notion database and retrieves the comments associated with each page.

- **Nodes Involved:**  
  - List database pages  
  - List comments

- **Node Details:**

  - **List database pages**  
    - *Type:* Notion Node  
    - *Role:* Retrieves all pages from the specified Notion database.  
    - *Configuration:*  
      - Operation: `getAll` database pages  
      - Database ID: Set dynamically from `notionDatabaseId` variable  
      - Credentials: Notion API account configured  
    - *Connections:* Outputs to "List comments".  
    - *Edge Cases:* API rate limits, empty database, or invalid credentials.

  - **List comments**  
    - *Type:* HTTP Request  
    - *Role:* Requests comments for each Notion page using Notion API's comments endpoint.  
    - *Configuration:*  
      - URL: `https://api.notion.com/v1/comments?block_id={{ $json.id }}` (dynamic block ID from page ID)  
      - Authentication: Predefined Notion API credentials  
    - *Connections:* Outputs to "Only keep pages that received new comments since last execution".  
    - *Edge Cases:* Empty comments list, API errors, block ID errors.

#### 2.3 Filtering

- **Overview:**  
  Filters pages to process only those that have new comments since the last execution, optimizing resource usage.

- **Nodes Involved:**  
  - Only keep pages that received new comments since last execution

- **Node Details:**

  - **Only keep pages that received new comments since last execution**  
    - *Type:* Filter Node  
    - *Role:* Checks if any comments or page edits are newer than the last recorded execution date property on the page.  
    - *Configuration:*  
      - Conditions:  
        - Checks if the last edited time of the latest comment is after the stored last execution date.  
        - Or if the last execution date is empty (first run or missing).  
    - *Connections:* Outputs to "Summarize conversation".  
    - *Edge Cases:* If date properties are missing or improperly formatted, filtering may fail or skip pages erroneously.

#### 2.4 AI Summarization

- **Overview:**  
  Summarizes the comment conversations per Notion page using an AI language model.

- **Nodes Involved:**  
  - Summarize conversation  
  - Google Gemini Chat Model  
  - Sticky Note2  
  - Sticky Note8

- **Node Details:**

  - **Summarize conversation**  
    - *Type:* LangChain Agent Node  
    - *Role:* Sends the JSON stringified comments to the AI model with a prompt requesting a summary of the conversation in up to six sentences.  
    - *Configuration:*  
      - Text: JSON string of comments with an explicit prompt to summarize and handle empty comments with "Empty" response.  
      - Prompt type: defined prompt (custom).  
    - *Connections:* Outputs AI summary text to "Add summary to Notion page and update last execution date".  
    - *Edge Cases:*  
      - If comments are empty, summary returns "Empty".  
      - Large comment size may affect token limits or latency.  
      - AI model errors or quota limitations.

  - **Google Gemini Chat Model**  
    - *Type:* Language Model Node (Google Gemini via LangChain)  
    - *Role:* Executes the AI language model call as backend to the agent node.  
    - *Configuration:*  
      - Model: `models/gemini-2.5-pro-preview-03-25`  
      - Credentials: Google Gemini (PaLM) API account  
    - *Connections:* Serves as AI model for "Summarize conversation".  
    - *Edge Cases:* Authentication errors, API rate limits, or model unavailability.

  - **Sticky Note2** and **Sticky Note8**  
    - *Type:* Sticky Note  
    - *Role:* Provide tips for prompt customization and API key setup for AI models.  
    - *Content Highlights:*  
      - Adapt the prompt for better results.  
      - Set API key for chosen LLM (OpenAI, Claude, Gemini, etc.).

#### 2.5 Updating Notion

- **Overview:**  
  This block writes the AI-generated summary back into the Notion page and updates the last execution date property.

- **Nodes Involved:**  
  - Add summary to Notion page and update last execution date

- **Node Details:**

  - **Add summary to Notion page and update last execution date**  
    - *Type:* Notion Node  
    - *Role:* Updates the Notion page's AI summary property with the generated summary and sets the last execution date property to the current workflow run timestamp.  
    - *Configuration:*  
      - Page ID: Dynamic from current Notion page ID  
      - Properties updated:  
        - AI Summary property (rich text) with the AI output  
        - Last Execution property (date) with timestamp from the "Run every hour" trigger  
      - Credentials: Notion API account  
    - *Connections:* None (end node).  
    - *Edge Cases:*  
      - Write failures due to permission errors or API limits.  
      - Missing properties cause update errors.  
      - Timestamp mismatch if workflow is manually triggered.

---

### 3. Summary Table

| Node Name                         | Node Type                            | Functional Role                                 | Input Node(s)                               | Output Node(s)                               | Sticky Note                                                                                       |
|----------------------------------|------------------------------------|------------------------------------------------|---------------------------------------------|----------------------------------------------|-------------------------------------------------------------------------------------------------|
| Run every hour                   | Schedule Trigger                   | Triggers workflow every hour                    | —                                           | Define your Notion Database                   |                                                                                                 |
| Define your Notion Database      | Set                               | Defines Notion database ID and property names  | Run every hour                              | List database pages                           | Step 1: Define variables (database ID, AI summary property, last execution property)            |
| List database pages             | Notion                            | Retrieves all pages from Notion database        | Define your Notion Database                 | List comments                                 | Step 2: Set Notion credentials via internal integration                                         |
| List comments                   | HTTP Request                      | Retrieves comments for each Notion page         | List database pages                         | Only keep pages that received new comments   | Step 3: Re-use Notion credentials set in Step 2                                                 |
| Only keep pages that received new comments since last execution | Filter                            | Filters pages with new comments since last run | List comments                              | Summarize conversation                        |                                                                                                 |
| Summarize conversation          | LangChain Agent                   | Summarizes comment conversations with AI       | Only keep pages that received new comments | Add summary to Notion page and update last execution date | Tip: Adapt prompt for better results                                                           |
| Google Gemini Chat Model        | Language Model (Google Gemini)   | Provides AI model for summarization             | Summarize conversation (as AI languageModel) | Summarize conversation (AI response)          | Step 4: Set API key for AI model (OpenAI, Claude, Gemini, etc.)                                |
| Add summary to Notion page and update last execution date | Notion                            | Updates Notion page with AI summary and timestamp | Summarize conversation                    | —                                            |                                                                                                 |
| Sticky Note                     | Sticky Note                      | Instructions and tips                            | —                                           | —                                            | Adapt template for webhooks, prompt tuning, token reduction, AI summary re-fill recommendations |
| Sticky Note5                   | Sticky Note                      | Step 1 instructions                              | —                                           | —                                            | Step 1: Define variables (database ID, summary prop, last exec prop)                            |
| Sticky Note6                   | Sticky Note                      | Step 2 instructions                              | —                                           | —                                            | Step 2: Set Notion credentials                                                                  |
| Sticky Note7                   | Sticky Note                      | Step 3 instructions                              | —                                           | —                                            | Step 3: Re-use Notion credentials                                                               |
| Sticky Note8                   | Sticky Note                      | Step 4 instructions                              | —                                           | —                                            | Step 4: Set API key for AI model                                                                |
| Sticky Note1                   | Sticky Note                      | Overview of workflow                             | —                                           | —                                            | Workflow summary with image                                                                     |
| Sticky Note2                   | Sticky Note                      | Prompt tip                                       | —                                           | —                                            | Tip: Adapt prompt for better results                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Schedule Trigger Node:**  
   - Name: `Run every hour`  
   - Type: Schedule Trigger  
   - Set to trigger every 1 hour.

2. **Create a Set Node:**  
   - Name: `Define your Notion Database`  
   - Define three string variables:  
     - `notionDatabaseId` = `"20d45c70c57381f09418d42c78ad360b"` (replace with your database ID)  
     - `AI Summary Notion Property (type text)` = `"AI Summary"` (replace with your text property name)  
     - `Last Execution Notion Property (type date)` = `"Last AI summary update"` (replace with your date property name)  
   - Connect output of `Run every hour` to this node.

3. **Create a Notion Node to List Database Pages:**  
   - Name: `List database pages`  
   - Resource: `databasePage`  
   - Operation: `getAll`  
   - Database ID: Expression referencing `notionDatabaseId` from previous node.  
   - Credentials: Connect your Notion internal integration account.  
   - Connect output of `Define your Notion Database` to this node.

4. **Create an HTTP Request Node to List Comments:**  
   - Name: `List comments`  
   - Method: GET  
   - URL: `https://api.notion.com/v1/comments?block_id={{ $json.id }}` (dynamic block ID)  
   - Authentication: Use the same Notion API credentials.  
   - Connect output of `List database pages` to this node.

5. **Create a Filter Node:**  
   - Name: `Only keep pages that received new comments since last execution`  
   - Condition 1 (OR):  
     - Left: Last edited time of last comment (`{{$json.results.length && $json.results[$json.results.length-1].last_edited_time}}`)  
     - Operation: after  
     - Right: Last execution date property from page (`{{ $('List database pages').item.json.properties[$('Define your Notion Database').item.json['Last Execution Notion Property']].date.start }}`)  
   - Condition 2 (OR):  
     - Left: Last execution date property (same as above)  
     - Operation: empty  
   - Connect output of `List comments` to this node.

6. **Create a LangChain Agent Node:**  
   - Name: `Summarize conversation`  
   - Text: Pass JSON stringified comments with prompt requesting a summary of max 6 sentences, returning "Empty" if no comments.  
   - Prompt Type: Define (custom prompt).  
   - Connect output of filter to this node.

7. **Create a Language Model Node (Google Gemini):**  
   - Name: `Google Gemini Chat Model`  
   - Model: `models/gemini-2.5-pro-preview-03-25`  
   - Credentials: Google Gemini (PaLM) API configured.  
   - Connect as AI model input to the LangChain Agent node.

8. **Create a Notion Node to Update the Page:**  
   - Name: `Add summary to Notion page and update last execution date`  
   - Resource: `databasePage`  
   - Operation: `update`  
   - Page ID: Dynamic from current page ID (`{{ $('List database pages').item.json.id }}`)  
   - Properties to update:  
     - AI Summary property (rich_text) with AI output (`{{ $json.output }}`)  
     - Last Execution property (date) with timestamp from `Run every hour` trigger (`{{ $('Run every hour').item.json.timestamp }}`)  
   - Credentials: Notion API account.  
   - Connect output of `Summarize conversation` to this node.

9. **(Optional) Add Sticky Notes:**  
   - Add explanatory sticky notes for each step, including instructions about credentials, variables, and prompt customization.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                   | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Instead of a cron trigger, use Notion webhooks to trigger the workflow when new comments are added for efficiency, especially in Cloud n8n environments.                                                                                       | Sticky Note: Adapt this template to your needs                                                     |
| To create Notion internal integrations and obtain credentials, visit https://www.notion.so/profile/integrations                                                                                                                               | Sticky Note6                                                                                        |
| For the AI summarization, various LLMs can be used such as OpenAI, Claude, or Google Gemini. Adjust the prompt to suit your needs for summary length and style.                                                                               | Sticky Note2, Sticky Note8                                                                          |
| If the AI summary is deleted manually, ensure the workflow can refill it by not filtering out pages with empty summaries or by running the workflow on demand.                                                                                 | Sticky Note: Adapt this template                                                                    |
| This workflow was designed to efficiently reduce token consumption by only sending necessary comment data to the AI and filtering pages without new comments since the last run.                                                               | Sticky Note: Adapt this template                                                                    |
| Example summary prompt included encourages concise summaries up to 6 sentences with a fallback "Empty" if no comments exist.                                                                                                                   | Node: Summarize conversation                                                                        |
| Workflow image preview: ![AI Summary of the comments](https://lh3.googleusercontent.com/d/1Zi9LqhFG1iU3q4YAM1v2DN-V7xl-eJjo)                                                                                                                   | Sticky Note1                                                                                       |

---

*Disclaimer: The text provided originates exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected material. All data handled are legal and public.*