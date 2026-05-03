Generate Personalized Language Learning News Digests with LLaMA-3.1 & DeepSeek AI

https://n8nworkflows.xyz/workflows/generate-personalized-language-learning-news-digests-with-llama-3-1---deepseek-ai-5660


# Generate Personalized Language Learning News Digests with LLaMA-3.1 & DeepSeek AI

---
### 1. Workflow Overview

This workflow automates the generation and personalized delivery of multilingual daily news digests tailored for language learners. It leverages a combination of scheduled triggers, data retrieval from Google Sheets, AI-powered content processing with LLaMA-3.1 and DeepSeek AI agents, and sends the final personalized news digest emails via Gmail.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Trigger & Data Retrieval**: Initiates workflow execution daily and fetches user data from Google Sheets.
- **1.2 Batch Processing of User Data**: Splits the data into manageable batches for processing.
- **1.3 Data Preparation & Enrichment**: Executes custom code and HTTP requests for data transformation and enrichment.
- **1.4 AI Content Generation**: Utilizes AI agents and OpenAI Chat model for personalized news digest creation.
- **1.5 Structured Output Parsing**: Parses AI-generated content into structured formats.
- **1.6 Email Dispatch**: Sends the finalized personalized news digest emails to users via Gmail.

---

### 2. Block-by-Block Analysis

#### 2.1 Scheduled Trigger & Data Retrieval

- **Overview:**  
  This block initiates the workflow daily and retrieves user information from a Google Sheets document, which likely contains user preferences, languages, or subscription details.

- **Nodes Involved:**  
  - Daily Trigger  
  - Google Sheets

- **Node Details:**

  - **Daily Trigger**  
    - *Type & Role:* Cron node; triggers workflow execution on a daily schedule.  
    - *Configuration:* Default daily scheduling without additional parameters specified.  
    - *Input/Output:* No input; output triggers Google Sheets node.  
    - *Edge Cases:* Misconfigured cron schedules could cause missed or repeated runs.

  - **Google Sheets**  
    - *Type & Role:* Retrieves data from a specified Google Sheets spreadsheet.  
    - *Configuration:* Parameters (spreadsheet ID, sheet name, range) not explicitly provided but expected to target user subscription data.  
    - *Input/Output:* Input from Daily Trigger; outputs rows of user data to Loop Over Items.  
    - *Edge Cases:* Authentication errors, rate limits, or empty data sets may cause failures.

#### 2.2 Batch Processing of User Data

- **Overview:**  
  Processes user data in batches to optimize performance and avoid timeouts or API rate limits.

- **Nodes Involved:**  
  - Loop Over Items

- **Node Details:**

  - **Loop Over Items**  
    - *Type & Role:* Splits input data into batches for sequential processing.  
    - *Configuration:* Default batch size (not explicitly set), allowing controlled iteration over user data.  
    - *Input/Output:* Inputs rows from Google Sheets; outputs batches to two separate branches: one to Code node and one empty branch.  
    - *Edge Cases:* Improper batch size may cause inefficient processing or timeouts.

#### 2.3 Data Preparation & Enrichment

- **Overview:**  
  Runs custom JavaScript code on each batch and performs HTTP requests to external services for data enrichment or fetching additional content.

- **Nodes Involved:**  
  - Code  
  - HTTP Request  
  - Edit Fields

- **Node Details:**

  - **Code**  
    - *Type & Role:* Custom JavaScript execution for data transformation or preparation.  
    - *Configuration:* User-defined code (contents not provided) likely processes user batch data to format or enrich it for the next step.  
    - *Input/Output:* Receives batch data from Loop Over Items; outputs transformed data to HTTP Request.  
    - *Edge Cases:* Runtime errors, invalid code logic, or unexpected input formats.

  - **HTTP Request**  
    - *Type & Role:* Calls external APIs or services, possibly to fetch news articles or metadata relevant to users' language preferences.  
    - *Configuration:* Not specified; expected to be a GET or POST request to a news or content API.  
    - *Input/Output:* Takes processed data from Code node; outputs API response to Edit Fields.  
    - *Edge Cases:* Network errors, API authentication failures, rate limiting, or unexpected response formats.

  - **Edit Fields**  
    - *Type & Role:* Sets or modifies data fields to prepare input for AI agents.  
    - *Configuration:* Likely restructures or filters HTTP response data for AI consumption.  
    - *Input/Output:* Input from HTTP Request; outputs to AI Agent.  
    - *Edge Cases:* Field misconfiguration or data type mismatches.

#### 2.4 AI Content Generation

- **Overview:**  
  Uses an AI Agent based on LLaMA-3.1 and DeepSeek AI integrated with OpenAI’s Chat model, to generate personalized multilingual news digests.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Structured Output Parser

- **Node Details:**

  - **AI Agent**  
    - *Type & Role:* Orchestrates AI language model invocation and output parsing, managing prompt construction and result handling.  
    - *Configuration:* Uses OpenAI Chat Model as the language model and Structured Output Parser for parsing outputs.  
    - *Input/Output:* Receives prepared data from Edit Fields; outputs AI-generated content to Gmail node.  
    - *Version:* Uses Langchain agent version 1.9.  
    - *Edge Cases:* API quota exhaustion, malformed prompts, timeouts.

  - **OpenAI Chat Model**  
    - *Type & Role:* Provides chat-based language model capabilities for content generation.  
    - *Configuration:* OpenAI credentials must be configured; parameters such as model name, temperature, and max tokens are expected but unspecified.  
    - *Input/Output:* Connected as AI Agent’s ai_languageModel input.  
    - *Edge Cases:* Authentication failures, rate limits, model unavailability.

  - **Structured Output Parser**  
    - *Type & Role:* Parses AI-generated text into structured JSON or other machine-readable formats to enable further processing.  
    - *Configuration:* Parser settings to extract specific fields from AI output.  
    - *Input/Output:* Connected as AI Agent’s ai_outputParser input.  
    - *Edge Cases:* Parsing failures due to unexpected AI output formats.

#### 2.5 Email Dispatch

- **Overview:**  
  Sends the personalized news digest emails to each user via Gmail, iterating over generated content per user.

- **Nodes Involved:**  
  - Gmail

- **Node Details:**

  - **Gmail**  
    - *Type & Role:* Sends emails using Gmail API with OAuth2 authentication.  
    - *Configuration:* Requires Gmail OAuth2 credentials; email content dynamically populated from AI Agent output.  
    - *Input/Output:* Input from AI Agent; outputs to Loop Over Items to continue batch processing if needed.  
    - *Edge Cases:* Authentication errors, quota limits, invalid email addresses, or connectivity issues.

---

### 3. Summary Table

| Node Name            | Node Type                                | Functional Role                        | Input Node(s)          | Output Node(s)        | Sticky Note                                                                                         |
|----------------------|-----------------------------------------|-------------------------------------|-----------------------|-----------------------|---------------------------------------------------------------------------------------------------|
| Daily Trigger        | Cron                                    | Initiate daily workflow execution   | -                     | Google Sheets          |                                                                                                   |
| Google Sheets        | Google Sheets                           | Retrieve user data from spreadsheet | Daily Trigger          | Loop Over Items        |                                                                                                   |
| Loop Over Items      | SplitInBatches                         | Batch processing of user data       | Google Sheets, Gmail   | Code (main), (empty)   |                                                                                                   |
| Code                 | Code                                    | Data transformation/enrichment      | Loop Over Items        | HTTP Request           |                                                                                                   |
| HTTP Request         | HTTP Request                           | Fetch external API data             | Code                   | Edit Fields            |                                                                                                   |
| Edit Fields          | Set                                     | Prepare data for AI processing      | HTTP Request            | AI Agent               |                                                                                                   |
| AI Agent             | Langchain Agent                        | AI content generation orchestration | Edit Fields             | Gmail                  |                                                                                                   |
| OpenAI Chat Model    | Langchain OpenAI Chat Model            | Language model for AI Agent         | - (ai_languageModel)   | AI Agent               |                                                                                                   |
| Structured Output Parser | Langchain Structured Output Parser | Parse AI output into structured data | - (ai_outputParser)    | AI Agent               |                                                                                                   |
| Gmail                | Gmail                                  | Send personalized emails            | AI Agent                | Loop Over Items        |                                                                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Cron Node (Daily Trigger):**  
   - Set to trigger the workflow once daily at a desired time. No additional parameters needed.

2. **Add Google Sheets Node:**  
   - Configure credentials for Google Sheets API with access to the target spreadsheet.  
   - Specify the spreadsheet ID and sheet name/range containing user subscription data.

3. **Insert SplitInBatches Node (Loop Over Items):**  
   - Connect Google Sheets node output to this node.  
   - Configure batch size (default is acceptable but can be adjusted based on expected data volume).

4. **Add Code Node:**  
   - Connect the first output of Loop Over Items to Code node.  
   - Insert JavaScript code to transform or enrich the batch data as needed (e.g., filtering, formatting).  
   - Ensure error handling in code to prevent workflow interruption.

5. **Add HTTP Request Node:**  
   - Connect Code node output to HTTP Request node.  
   - Configure with API endpoint(s) to fetch news or related content.  
   - Set method (GET/POST), headers (including authentication if required), and payload accordingly.

6. **Add Set Node (Edit Fields):**  
   - Connect HTTP Request output to Set node.  
   - Configure fields to reshape the data, extracting relevant information for AI consumption.

7. **Add Langchain AI Agent Node:**  
   - Connect Set node output to AI Agent node.  
   - Configure AI Agent to use OpenAI Chat Model and Structured Output Parser nodes as inputs.

8. **Add Langchain OpenAI Chat Model Node:**  
   - Add separately and configure with OpenAI credentials.  
   - Set model parameters such as model ID (e.g., GPT-4 or LLaMA-3.1 if accessible), temperature, max tokens.

9. **Add Langchain Structured Output Parser Node:**  
   - Configure parsing rules/templates to convert AI output into structured JSON.

10. **Add Gmail Node:**  
    - Connect AI Agent output to Gmail node.  
    - Configure Gmail OAuth2 credentials with send email permission.  
    - Set email recipient dynamically from user data, subject line, and email body from AI-generated content.  
    - Handle fallback or error email addresses as needed.

11. **Connect Gmail Node Output Back to Loop Over Items (Second Output):**  
    - To manage continuation or finalization of batch processing.

12. **Final Testing:**  
    - Run the workflow in test mode to verify each step’s output.  
    - Monitor logs for errors like authentication failures, API limits, or parsing issues.  
    - Adjust batch sizes or add error handling nodes if needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                   | Context or Link                                                                                 |
|---------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| The workflow integrates LLaMA-3.1 & DeepSeek AI via Langchain nodes for personalized multilingual content.    | Requires Langchain nodes and appropriate AI credentials configured in n8n.                     |
| Gmail node requires OAuth2 credentials with proper scopes enabled for sending emails on behalf of the user.   | See https://developers.google.com/identity/protocols/oauth2 for Gmail OAuth2 setup.             |
| Google Sheets node needs API access and sharing permissions for the target spreadsheet to the service account. | Refer to https://developers.google.com/sheets/api/quickstart/nodejs for setup details.          |
| Batch processing is used to optimize handling of large user lists and avoid rate limits or timeouts.          | Adjust batch size in SplitInBatches node as per data volume and API rate limits.                 |
| Custom JavaScript code node must be carefully maintained for changes in data structure or API responses.      | Use try-catch blocks to handle unexpected input data and avoid workflow failures.               |

---

**Disclaimer:** The text provided is exclusively derived from an n8n automated workflow. All data processed is legal and publicly available; content respects applicable content policies with no illegal or offensive elements.