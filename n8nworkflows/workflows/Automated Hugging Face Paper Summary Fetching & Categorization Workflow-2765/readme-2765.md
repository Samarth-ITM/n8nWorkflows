Automated Hugging Face Paper Summary Fetching & Categorization Workflow

https://n8nworkflows.xyz/workflows/automated-hugging-face-paper-summary-fetching---categorization-workflow-2765


# Automated Hugging Face Paper Summary Fetching & Categorization Workflow

### 1. Workflow Overview

This workflow automates the daily retrieval, analysis, and categorization of the latest research papers from Hugging Face, integrating with Notion for storage and Slack for notifications. It is designed for researchers, knowledge managers, or teams who want to stay updated on new papers without manual effort.

The workflow is logically divided into the following blocks:

- **1.1 Scheduled Fetching:** Automatically triggers every weekday at 8 AM to fetch the latest papers from Hugging Face.
- **1.2 Paper Extraction and Splitting:** Parses the fetched data and splits it into individual paper items for processing.
- **1.3 Duplication Check:** Queries Notion to verify if a paper has already been stored, preventing duplicates.
- **1.4 Detailed Paper Retrieval and Abstract Extraction:** For new papers, fetches detailed content and extracts the abstract.
- **1.5 AI Analysis:** Uses OpenAI to analyze the abstract, extract insights, and categorize the paper.
- **1.6 Data Storage and Notification:** Stores the analyzed summary in Notion and sends a notification with paper details to Slack.

---

### 2. Block-by-Block Analysis

#### 1.1 Scheduled Fetching

- **Overview:**  
  This block triggers the workflow automatically every weekday at 8 AM and initiates the fetching of the latest papers from Hugging Face.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Request Hugging Face Paper  
  - Extract Hugging Face Paper  

- **Node Details:**

  - **Schedule Trigger**  
    - Type: Schedule Trigger  
    - Role: Starts the workflow on a schedule (weekday 8 AM)  
    - Configuration: Default schedule node configured for weekdays at 8 AM (implied by description)  
    - Inputs: None (trigger node)  
    - Outputs: Connects to "Request Hugging Face Paper"  
    - Edge Cases: Misconfiguration of schedule could cause missed or repeated triggers  

  - **Request Hugging Face Paper**  
    - Type: HTTP Request  
    - Role: Fetches the latest papers from Hugging Face API or website  
    - Configuration: HTTP GET request to Hugging Face papers endpoint (exact URL not specified)  
    - Inputs: Trigger from Schedule Trigger  
    - Outputs: Raw HTML or JSON response to "Extract Hugging Face Paper"  
    - Edge Cases: Network errors, API rate limits, unexpected response format  

  - **Extract Hugging Face Paper**  
    - Type: HTML Extract  
    - Role: Parses the HTTP response to extract paper listings  
    - Configuration: Uses CSS selectors or XPath to extract paper data (e.g., titles, URLs)  
    - Inputs: HTTP response from previous node  
    - Outputs: Parsed paper list to "Split Out"  
    - Edge Cases: Changes in webpage structure may break extraction  

#### 1.2 Paper Extraction and Splitting

- **Overview:**  
  Splits the list of papers into individual items for sequential processing.

- **Nodes Involved:**  
  - Split Out  
  - Loop Over Items  

- **Node Details:**

  - **Split Out**  
    - Type: Split Out  
    - Role: Converts the extracted list into an array of individual paper items  
    - Configuration: Default, splitting the array of papers  
    - Inputs: Extracted papers from "Extract Hugging Face Paper"  
    - Outputs: Individual paper items to "Loop Over Items"  
    - Edge Cases: Empty or malformed input array  

  - **Loop Over Items**  
    - Type: Split In Batches  
    - Role: Processes each paper item one by one to avoid overloading downstream nodes  
    - Configuration: Batch size likely set to 1 (default)  
    - Inputs: Individual paper items from "Split Out"  
    - Outputs: Each paper item to "Check Paper URL Existed"  
    - Edge Cases: Large number of papers may slow workflow; batch size misconfiguration  

#### 1.3 Duplication Check

- **Overview:**  
  Checks if the paper URL already exists in the Notion database to avoid duplicate entries.

- **Nodes Involved:**  
  - Check Paper URL Existed  
  - If  

- **Node Details:**

  - **Check Paper URL Existed**  
    - Type: Notion  
    - Role: Queries Notion database for existing paper URL  
    - Configuration: Search filter on Notion database by paper URL property  
    - Inputs: Single paper item from "Loop Over Items"  
    - Outputs: Query result to "If" node  
    - Edge Cases: Notion API rate limits, authentication errors, missing database or property  

  - **If**  
    - Type: If  
    - Role: Conditional branching based on whether the paper exists in Notion  
    - Configuration: Checks if query result is empty (paper not found) or not  
    - Inputs: Query result from "Check Paper URL Existed"  
    - Outputs:  
      - True branch: Paper exists → loops back to "Loop Over Items" (skips processing)  
      - False branch: Paper does not exist → proceeds to "Request Hugging Face Paper Detail"  
    - Edge Cases: Expression evaluation errors, unexpected query results  

#### 1.4 Detailed Paper Retrieval and Abstract Extraction

- **Overview:**  
  For new papers, fetches detailed paper content and extracts the abstract for analysis.

- **Nodes Involved:**  
  - Request Hugging Face Paper Detail  
  - Extract Hugging Face Paper Abstract  

- **Node Details:**

  - **Request Hugging Face Paper Detail**  
    - Type: HTTP Request  
    - Role: Fetches detailed page or API endpoint for the specific paper  
    - Configuration: HTTP GET request using paper URL from previous node  
    - Inputs: Paper URL from "If" node false branch  
    - Outputs: Detailed paper HTML or JSON to "Extract Hugging Face Paper Abstract"  
    - Edge Cases: Network errors, invalid URLs, API changes  

  - **Extract Hugging Face Paper Abstract**  
    - Type: HTML Extract  
    - Role: Parses detailed paper content to extract the abstract text  
    - Configuration: CSS selector or XPath targeting abstract section  
    - Inputs: HTTP response from previous node  
    - Outputs: Abstract text to "OpenAI Analysis Abstract"  
    - Edge Cases: Changes in page structure, missing abstract  

#### 1.5 AI Analysis

- **Overview:**  
  Uses OpenAI to analyze the abstract, extract key insights, and categorize the paper content.

- **Nodes Involved:**  
  - OpenAI Analysis Abstract  

- **Node Details:**

  - **OpenAI Analysis Abstract**  
    - Type: OpenAI (LangChain integration)  
    - Role: Sends abstract text to OpenAI for analysis and categorization  
    - Configuration:  
      - Model: Likely GPT-4 or GPT-3.5 (not explicitly stated)  
      - Prompt: Custom prompt to extract insights and categorize content  
    - Inputs: Abstract text from "Extract Hugging Face Paper Abstract"  
    - Outputs: Analyzed summary and categories to both "Send Analysis Result Slack" and "Store Abstract Notion"  
    - Edge Cases: API rate limits, authentication errors, prompt failures, unexpected output format  

#### 1.6 Data Storage and Notification

- **Overview:**  
  Stores the analyzed paper summary in Notion and sends a notification with paper details to Slack.

- **Nodes Involved:**  
  - Store Abstract Notion  
  - Send Analysis Result Slack  

- **Node Details:**

  - **Store Abstract Notion**  
    - Type: Notion  
    - Role: Creates or updates a record in Notion with the paper summary and metadata  
    - Configuration: Maps fields such as title, URL, abstract, analysis results, and categories  
    - Inputs: Analysis output from "OpenAI Analysis Abstract"  
    - Outputs: Loops back to "Loop Over Items" to continue processing next paper  
    - Edge Cases: Notion API limits, authentication errors, schema mismatches  

  - **Send Analysis Result Slack**  
    - Type: Slack  
    - Role: Sends a message to a configured Slack channel with paper details and analysis summary  
    - Configuration: Uses Slack webhook or OAuth2 credentials; message formatting includes paper title, summary, and link  
    - Inputs: Analysis output from "OpenAI Analysis Abstract"  
    - Outputs: Loops back to "Loop Over Items"  
    - Edge Cases: Slack API errors, invalid webhook, message formatting issues  

---

### 3. Summary Table

| Node Name                     | Node Type                     | Functional Role                          | Input Node(s)                  | Output Node(s)                           | Sticky Note                                                                                              |
|-------------------------------|-------------------------------|----------------------------------------|-------------------------------|-----------------------------------------|--------------------------------------------------------------------------------------------------------|
| Schedule Trigger              | Schedule Trigger              | Initiates workflow on schedule         | None                          | Request Hugging Face Paper               |                                                                                                        |
| Request Hugging Face Paper    | HTTP Request                 | Fetches latest papers from Hugging Face| Schedule Trigger              | Extract Hugging Face Paper               |                                                                                                        |
| Extract Hugging Face Paper    | HTML Extract                 | Parses fetched papers list              | Request Hugging Face Paper    | Split Out                               |                                                                                                        |
| Split Out                    | Split Out                    | Splits paper list into individual items| Extract Hugging Face Paper    | Loop Over Items                         |                                                                                                        |
| Loop Over Items              | Split In Batches             | Processes papers one by one             | Split Out                    | Check Paper URL Existed                  |                                                                                                        |
| Check Paper URL Existed      | Notion                       | Checks for duplicate paper in Notion   | Loop Over Items              | If                                      |                                                                                                        |
| If                          | If                          | Branches based on duplication check    | Check Paper URL Existed       | Request Hugging Face Paper Detail (False branch), Loop Over Items (True branch) |                                                                                                        |
| Request Hugging Face Paper Detail | HTTP Request                 | Fetches detailed paper content          | If (False branch)            | Extract Hugging Face Paper Abstract      |                                                                                                        |
| Extract Hugging Face Paper Abstract | HTML Extract                 | Extracts abstract from paper detail     | Request Hugging Face Paper Detail | OpenAI Analysis Abstract                 |                                                                                                        |
| OpenAI Analysis Abstract     | OpenAI (LangChain)           | Analyzes abstract and categorizes paper| Extract Hugging Face Paper Abstract | Send Analysis Result Slack, Store Abstract Notion |                                                                                                        |
| Store Abstract Notion        | Notion                       | Stores analyzed summary in Notion      | OpenAI Analysis Abstract     | Loop Over Items                         |                                                                                                        |
| Send Analysis Result Slack   | Slack                        | Sends notification to Slack channel    | OpenAI Analysis Abstract     | Loop Over Items                         |                                                                                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Type: Schedule Trigger  
   - Configure to run every weekday at 8 AM (e.g., Monday to Friday, 08:00)  
   - No credentials required  

2. **Create HTTP Request Node: Request Hugging Face Paper**  
   - Connect from Schedule Trigger  
   - Method: GET  
   - URL: Hugging Face papers listing endpoint (e.g., https://huggingface.co/papers or API endpoint)  
   - Authentication: None or as required by Hugging Face  
   - Response Format: HTML or JSON depending on endpoint  

3. **Create HTML Extract Node: Extract Hugging Face Paper**  
   - Connect from Request Hugging Face Paper  
   - Configure extraction rules (CSS selectors or XPath) to extract paper titles, URLs, and summaries from response  
   - Output: Array of paper objects  

4. **Create Split Out Node**  
   - Connect from Extract Hugging Face Paper  
   - Configure to split the array of papers into individual items  

5. **Create Split In Batches Node: Loop Over Items**  
   - Connect from Split Out  
   - Batch size: 1 (process one paper at a time)  

6. **Create Notion Node: Check Paper URL Existed**  
   - Connect from Loop Over Items  
   - Operation: Search database  
   - Database: Your Notion database for papers  
   - Filter: Paper URL equals current item's URL  
   - Credentials: Notion OAuth2 or integration token  

7. **Create If Node**  
   - Connect from Check Paper URL Existed  
   - Condition: If search result is empty (paper not found) → False branch; else → True branch  
   - True branch connects back to Loop Over Items (skip processing)  
   - False branch proceeds to next step  

8. **Create HTTP Request Node: Request Hugging Face Paper Detail**  
   - Connect from If node False branch  
   - Method: GET  
   - URL: Paper URL from current item  
   - Response Format: HTML or JSON  

9. **Create HTML Extract Node: Extract Hugging Face Paper Abstract**  
   - Connect from Request Hugging Face Paper Detail  
   - Configure extraction to get abstract text from detailed paper page  

10. **Create OpenAI Node: OpenAI Analysis Abstract**  
    - Connect from Extract Hugging Face Paper Abstract  
    - Credentials: OpenAI API key  
    - Model: GPT-4 or GPT-3.5  
    - Prompt: Custom prompt to analyze abstract, extract insights, and categorize  
    - Input: Abstract text  

11. **Create Slack Node: Send Analysis Result Slack**  
    - Connect from OpenAI Analysis Abstract  
    - Credentials: Slack OAuth2 or webhook URL  
    - Channel: Designated Slack channel  
    - Message: Format with paper title, summary, and link  

12. **Create Notion Node: Store Abstract Notion**  
    - Connect from OpenAI Analysis Abstract  
    - Operation: Create or update page in Notion database  
    - Map fields: title, URL, abstract, analysis results, categories  
    - Credentials: Notion OAuth2 or integration token  

13. **Connect Store Abstract Notion and Send Analysis Result Slack nodes back to Loop Over Items**  
    - This allows processing of next paper in batch  

14. **Activate the workflow**  
    - Ensure all credentials are configured and tested  
    - Enable the workflow to run automatically on schedule  

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                  |
|-----------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| Register for an n8n account and use the n8n cloud service for easy setup.                                              | https://n8n.cloud                                                                                |
| Connect OpenAI, Notion, and Slack accounts with appropriate API tokens before activating the workflow.                | Setup takes approximately 10–15 minutes                                                        |
| Import the workflow template to streamline setup and avoid manual configuration errors.                               | n8n workflow import feature                                                                     |
| The workflow triggers every weekday at 8 AM to fetch the latest Hugging Face papers automatically.                     | Scheduled Fetching block                                                                        |
| Slack notifications include paper details and analysis for quick team updates.                                         | Slack node configured with webhook or OAuth2                                                    |
| Notion database schema should include properties for paper URL, title, abstract, and analysis results for compatibility.| Notion integration requires proper database setup                                              |
| Potential failure points include API rate limits, authentication errors, and changes in Hugging Face page structure.  | Monitor logs and update extraction selectors as needed                                         |
| For more information on n8n and integrations, visit the official documentation.                                        | https://docs.n8n.io/                                                                            |

---

This document provides a complete, structured reference to understand, reproduce, and maintain the "Automated Hugging Face Paper Summary Fetching & Categorization Workflow" in n8n.