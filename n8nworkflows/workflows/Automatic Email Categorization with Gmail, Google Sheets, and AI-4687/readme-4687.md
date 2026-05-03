Automatic Email Categorization with Gmail, Google Sheets, and AI

https://n8nworkflows.xyz/workflows/automatic-email-categorization-with-gmail--google-sheets--and-ai-4687


# Automatic Email Categorization with Gmail, Google Sheets, and AI

### 1. Workflow Overview

This workflow automates the categorization of Gmail emails by leveraging Google Sheets for category definitions and an AI model to assign the best matching category label to each email thread. It is designed for users who want to organize their Gmail inbox automatically based on custom categories defined in a Google Sheet, improving email management and productivity.

The workflow is logically divided into the following blocks:

- **1.1 Configuration Setup:** Defines key parameters such as the Google Sheets URL containing category labels, email filter criteria, and processing limits.
- **1.2 Email Retrieval:** Fetches unlabeled Gmail threads based on the criteria set.
- **1.3 Category Labels Retrieval:** Reads category definitions from Google Sheets.
- **1.4 AI Categorization:** Uses an AI agent, powered by an OpenRouter model, to assign the most appropriate category name to each email based on its subject and content.
- **1.5 Label Management and Assignment:** Creates Gmail labels if they do not exist, retrieves existing labels, and applies the AI-determined label to the corresponding email thread.
- **1.6 Control and Looping:** Processes emails individually using batching and looping nodes to handle multiple emails per execution.

---

### 2. Block-by-Block Analysis

---

#### 2.1 Configuration Setup

- **Overview:**  
Sets up static and dynamic configuration parameters used throughout the workflow, including the Google Sheets URL for category labels, Gmail search filters, and the limit on emails to process per run.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Config  
  - Sticky Note (Config Instructions)  
  - Sticky Note (Parameter Explanation)

- **Node Details:**

| Node Name       | Details                                                                                                   |
|-----------------|-----------------------------------------------------------------------------------------------------------|
| Schedule Trigger| Type: Schedule Trigger<br>Triggers the workflow every hour (configurable interval).<br>Output: starts workflow execution.<br>Failures may occur if the n8n instance is down or time zone mismatches. |
| Config          | Type: Set Node<br>Stores: `sheets_url` (URL string to Google Sheets with category labels), `extra_filter` (optional Gmail search query filter), `limit` (max emails per run, default 2).<br>Inputs: from Schedule Trigger.<br>Outputs: to Get Labels.<br>Expression usage to dynamically fetch values.<br>Failure possible if URL is malformed or parameters are missing. |
| Sticky Note (Config Instructions) | Type: Sticky Note<br>Provides instructions and a link to a sample Google Sheet to duplicate.<br>Contextual help for users configuring the workflow. |
| Sticky Note (Parameter Explanation) | Type: Sticky Note<br>Explains the meaning of each Config parameter to end users. |

---

#### 2.2 Category Labels Retrieval

- **Overview:**  
Fetches category labels and their definitions from the specified Google Sheet URL. These categories are used by the AI to classify emails.

- **Nodes Involved:**  
  - Get Labels  

- **Node Details:**

| Node Name  | Details                                                                                       |
|------------|-----------------------------------------------------------------------------------------------|
| Get Labels | Type: Google Sheets Node<br>Reads data from the sheet URL provided in `sheets_url` parameter.<br>Uses OAuth2 credentials for authentication.<br>Outputs an array of category definitions.<br>Failure points include invalid credentials, sheet URL errors, or permission issues. |

---

#### 2.3 Email Retrieval

- **Overview:**  
Retrieves Gmail threads that are unlabeled and match optional filter criteria, limited by the configured maximum count.

- **Nodes Involved:**  
  - Get Messages  

- **Node Details:**

| Node Name   | Details                                                                                              |
|-------------|----------------------------------------------------------------------------------------------------|
| Get Messages| Type: Gmail Node<br>Fetches threads without user labels (`has:nouserlabels`) using optional filters.<br>Limit set dynamically from `Config.limit`.<br>Uses OAuth2 credentials for Gmail.<br>Potential failures: authentication issues, quota limits, or malformed Gmail queries.<br>Output: emails with headers and text content for AI analysis. |

---

#### 2.4 AI Categorization

- **Overview:**  
Analyzes each email’s subject and body text, then uses an AI model to select the best matching category name from the Google Sheets categories.

- **Nodes Involved:**  
  - AI Agent  
  - OpenRouter Chat Model  

- **Node Details:**

| Node Name        | Details                                                                                                                  |
|------------------|--------------------------------------------------------------------------------------------------------------------------|
| AI Agent         | Type: Langchain Agent Node<br>Inputs: email subject and text, plus category definitions from Get Labels.<br>Prompt instructs the AI to output only one exact category name.<br>Uses expression to dynamically format prompt with email and categories.<br>Outputs the chosen category name.<br>Failures: AI service errors, prompt formatting issues, or unexpected AI responses.<br>Version-specific: requires Langchain integration. |
| OpenRouter Chat Model | Type: Langchain OpenRouter Model<br>Configured with a free or paid OpenRouter model, e.g., "deepseek/deepseek-r1:free".<br>Requires OpenRouter API credentials.<br>Acts as the AI backend for the AI Agent node.<br>Failures: API key errors, model unavailability, network issues. |

---

#### 2.5 Label Management and Assignment

- **Overview:**  
Ensures Gmail labels exist for the AI-assigned categories, retrieves existing labels, filters to find the matching label, and applies it to the email thread.

- **Nodes Involved:**  
  - Loop Over Items  
  - Fields For Loop  
  - Create Label if Doesn't exist  
  - Get Existing Labels  
  - Filter  
  - Gmail1  

- **Node Details:**

| Node Name               | Details                                                                                                          |
|-------------------------|------------------------------------------------------------------------------------------------------------------|
| Loop Over Items         | Type: SplitInBatches Node<br>Processes emails one by one (or in batches) to handle labeling sequentially.<br>Inputs: from Gmail1 or Fields For Loop.<br>Outputs: to Create Label node or ends loop.<br>Failures: batch processing errors, data loss if interrupted. |
| Fields For Loop         | Type: Set Node<br>Extracts and maps necessary fields (`threadId` and AI output label) for use in subsequent nodes.<br>Input: from AI Agent.<br>Output: to Loop Over Items.<br>Failures: expression errors or missing fields. |
| Create Label if Doesn't exist | Type: Gmail Node<br>Creates a Gmail label named after the AI output if it does not exist.<br>On error, continues without stopping workflow.<br>Input: from Loop Over Items.<br>Output: to Get Existing Labels.<br>Failures: permission issues, label naming conflicts. |
| Get Existing Labels     | Type: Gmail Node<br>Retrieves all user labels from Gmail.<br>Input: from Create Label.<br>Output: to Filter.<br>Failures: authentication or API rate limits. |
| Filter                  | Type: Filter Node<br>Matches the label name from Gmail labels with the AI output label.<br>Input: from Get Existing Labels.<br>Output: to Gmail1 if match found.<br>Failures: filtering logic errors or unmatched labels due to case sensitivity. |
| Gmail1                  | Type: Gmail Node<br>Adds the matched label to the Gmail thread.<br>Input: from Filter.<br>Uses `threadId` and label ID.<br>Failures: API errors, thread not found, or label application failures. |

---

#### 2.6 Control and Looping

- **Overview:**  
Manages execution flow by triggering on schedule, setting configuration, and controlling the batching and processing of emails step-by-step.

- **Nodes Involved:**  
  - Schedule Trigger  
  - Config  
  - Get Labels  
  - Get Messages  
  - AI Agent  
  - Fields For Loop  
  - Loop Over Items  

- **Node Details:**

| Node Name       | Details                                                                                     |
|-----------------|---------------------------------------------------------------------------------------------|
| Schedule Trigger| Triggers workflow hourly.                                                                   |
| Config          | Provides parameters to all downstream nodes.                                               |
| Get Labels      | Feeds category data to AI.                                                                 |
| Get Messages    | Provides emails for AI evaluation.                                                        |
| AI Agent        | Produces label classification.                                                            |
| Fields For Loop | Prepares data for batch processing.                                                       |
| Loop Over Items | Iterates over each email for label creation and assignment.                               |

---

### 3. Summary Table

| Node Name                 | Node Type                        | Functional Role                              | Input Node(s)                      | Output Node(s)                    | Sticky Note                                                                                     |
|---------------------------|---------------------------------|----------------------------------------------|----------------------------------|----------------------------------|------------------------------------------------------------------------------------------------|
| Schedule Trigger          | Schedule Trigger                 | Starts workflow periodically                  | -                                | Config                           |                                                                                                |
| Config                    | Set                             | Holds configuration parameters                | Schedule Trigger                 | Get Labels                      | ## 1 - Config 1) [G Sheets to Duplicate](https://docs.google.com/spreadsheets/d/1LKIx1Z3dCSX1uzyZH9s2HE0QRMvLTDI6sJApFU5LTj0/edit?gid=0#gid=0) 2) Copy URL in Config node ➡️➡️ |
| Sticky Note (Config)      | Sticky Note                     | User instructions for config                   | -                                | -                                | ## 1 - Config 1) [G Sheets to Duplicate](https://docs.google.com/spreadsheets/d/1LKIx1Z3dCSX1uzyZH9s2HE0QRMvLTDI6sJApFU5LTj0/edit?gid=0#gid=0) 2) Copy URL in Config node ➡️➡️ |
| Get Labels                | Google Sheets                   | Retrieves category labels                      | Config                          | Get Messages                   |                                                                                                |
| Get Messages              | Gmail                           | Fetches unlabeled emails                       | Get Labels                     | AI Agent                      |                                                                                                |
| AI Agent                  | Langchain Agent                 | Categorizes emails using AI                     | Get Messages                   | Fields For Loop               |                                                                                                |
| Fields For Loop           | Set                             | Prepares data for looping                       | AI Agent                       | Loop Over Items               |                                                                                                |
| Loop Over Items           | SplitInBatches                  | Processes emails one by one                      | Gmail1 / Fields For Loop        | Create Label if Doesn't exist / (empty) |                                                                                                |
| Create Label if Doesn't exist | Gmail                         | Creates Gmail label if missing                  | Loop Over Items                | Get Existing Labels           |                                                                                                |
| Get Existing Labels       | Gmail                           | Retrieves Gmail labels                          | Create Label if Doesn't exist | Filter                       |                                                                                                |
| Filter                    | Filter                          | Matches AI label with existing Gmail label     | Get Existing Labels            | Gmail1                       |                                                                                                |
| Gmail1                   | Gmail                           | Assigns label to email thread                   | Filter                        | Loop Over Items              |                                                                                                |
| Sticky Note1              | Sticky Note                     | Instructions for AI connection and model setup | -                              | -                            | ## 2 - Connect AI (Optional: find free model) 1) Get OpenRouter API Key: https://openrouter.ai/settings/keys 2) [List of free models](https://openrouter.ai/models?max_price=0&order=top-weekly) [Popular models](https://openrouter.ai/models?order=top-weekly) Best 2025 May: deepseek/deepseek-chat-v3-0324:free Paid recommended: gpt-4.1-mini 3) Go to OpenRouter Chat Model Node + Add Credential + Choose model |
| OpenRouter Chat Model     | Langchain OpenRouter Model      | AI language model backend                      | AI Agent                      | AI Agent                     | ## 2 - Connect AI (Optional: find free model) 1) Get OpenRouter API Key: https://openrouter.ai/settings/keys 2) [List of free models](https://openrouter.ai/models?max_price=0&order=top-weekly) [Popular models](https://openrouter.ai/models?order=top-weekly) Best 2025 May: deepseek/deepseek-chat-v3-0324:free Paid recommended: gpt-4.1-mini 3) Go to OpenRouter Chat Model Node + Add Credential + Choose model |
| Sticky Note4              | Sticky Note                     | Explanation of Config parameters               | -                              | -                            | 👆⬆️ 1 - sheets_url - Where the category definitions live 2 - extra_filter - Leave empty to process all emails. Use any gmail search filters. 3 - limit - max no. of emails to process in a run |
| Sticky Note2              | Sticky Note                     | Workflow setup header                           | -                              | -                            | # Setup                                                                                        |
| Sticky Note5              | Sticky Note                     | Author information and contact                  | -                              | -                            | # About the Author ![Milan](https://gravatar.com/avatar/95700d17ba300a9f14c1b8cacf933df7720027b3adda9cbe6183d89142925422?r=pg&d=retro&size=100) Milan - SmoothWork.ai We help businesses eliminate busywork by building compact business tools tailored to your process. ▶️ [YouTube Channel](https://www.youtube.com/@vasarmilan) 📞 [Book a Free Consulting Call](https://smoothwork.ai/book-a-call/) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node:**  
   - Type: Schedule Trigger  
   - Set to run every 1 hour (or desired interval) to trigger the workflow periodically.

2. **Create Config Node:**  
   - Type: Set  
   - Define three variables:  
     - `sheets_url` (string): URL to your Google Sheet with category labels (copy from sample or your own).  
     - `extra_filter` (string): Optional Gmail search filter query, leave empty for all unlabeled emails.  
     - `limit` (number): Max number of emails to process per run (default 2 for testing).

3. **Connect Schedule Trigger → Config.**

4. **Create Get Labels Node:**  
   - Type: Google Sheets  
   - Credentials: Authenticate with Google Sheets OAuth2.  
   - Configuration:  
     - Document ID and Sheet Name: Use expressions to extract these from `sheets_url` in Config node.  
   - Purpose: Read category definitions for AI input.

5. **Connect Config → Get Labels.**

6. **Create Get Messages Node:**  
   - Type: Gmail  
   - Credentials: Authenticate with Gmail OAuth2.  
   - Operation: Get All threads without user labels (`has:nouserlabels`) plus optional filter from `extra_filter`.  
   - Limit: Use expression from Config node `limit`.  
   - Output: Full email data including subject and body.

7. **Connect Get Labels → Get Messages.**

8. **Create OpenRouter Chat Model Node:**  
   - Type: Langchain OpenRouter Model  
   - Credentials: Add OpenRouter API key (get from https://openrouter.ai/settings/keys).  
   - Model: Select a free or paid model (e.g., `deepseek/deepseek-r1:free`).  
   - No additional parameters needed.

9. **Create AI Agent Node:**  
   - Type: Langchain Agent  
   - Configure prompt:  
     ```
     Email subject:
     {{ $json.headers.subject }}

     Text: {{ $json.text }}

     Choose the best fitting of the category names below, based on their definition:
     {{ JSON.stringify($('Get Labels').all()) }}

     Important: ONLY respond with the category Name. EXACTLY one of the names, do not add any other texts.

     So response should be one of:
     {{ $('Get Labels').all().map(({ json: { Name } }) => Name ).join('\n') }}
     ```  
   - Connect AI Agent's language model to OpenRouter Chat Model node.

10. **Connect Get Messages → AI Agent.**

11. **Create Fields For Loop Node:**  
    - Type: Set  
    - Assign two fields:  
      - `threadId` → `={{ $('Get Messages').item.json.threadId }}`  
      - `output` → `={{ $json.output }}` (AI classification result)

12. **Connect AI Agent → Fields For Loop.**

13. **Create Loop Over Items Node:**  
    - Type: SplitInBatches  
    - Processes each classified email sequentially.

14. **Connect Fields For Loop → Loop Over Items.**

15. **Create Create Label if Doesn't exist Node:**  
    - Type: Gmail  
    - Operation: Create Label  
    - Name: Use expression `={{ $json.output }}` (category name)  
    - On Error: Continue without stopping (to handle label already exists).  
    - Credentials: Gmail OAuth2.

16. **Connect Loop Over Items (main output) → Create Label if Doesn't exist.**

17. **Create Get Existing Labels Node:**  
    - Type: Gmail  
    - Operation: List all labels (returnAll = true).  
    - Credentials: Gmail OAuth2.

18. **Connect Create Label if Doesn't exist → Get Existing Labels.**

19. **Create Filter Node:**  
    - Condition: Check if Gmail label name equals AI output label name.  
      - Expression: LeftValue `={{ $json.name }}` equals RightValue `={{ $('Loop Over Items').item.json.output }}`  
    - Case-sensitive, strict comparison.

20. **Connect Get Existing Labels → Filter.**

21. **Create Gmail1 Node (Add Label to Thread):**  
    - Type: Gmail  
    - Operation: Add Labels to Thread  
    - Thread ID: `={{ $('Loop Over Items').item.json.threadId }}`  
    - Label IDs: `={{ $json.id }}` from Filter output  
    - Credentials: Gmail OAuth2.

22. **Connect Filter (true branch) → Gmail1.**

23. **Connect Gmail1 → Loop Over Items (to continue batch processing).**

---

### 5. General Notes & Resources

| Note Content                                                                                                     | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow requires setting up Google Sheets with category label definitions. Use provided sample sheet or create your own. | [Sample Google Sheet](https://docs.google.com/spreadsheets/d/1LKIx1Z3dCSX1uzyZH9s2HE0QRMvLTDI6sJApFU5LTj0/edit?gid=0#gid=0) |
| OpenRouter API key is required for AI categorization. Use free or paid models depending on your needs.          | https://openrouter.ai/settings/keys and https://openrouter.ai/models                              |
| Recommended OpenRouter models as of May 2025: `deepseek/deepseek-chat-v3-0324:free` (free), or paid like `gpt-4.1-mini`. | See Sticky Note1 content for details.                                                              |
| Author: Milan - SmoothWork.ai. For consulting or tutorials, visit YouTube or book a free consulting call.       | YouTube: https://www.youtube.com/@vasarmilan; Consulting: https://smoothwork.ai/book-a-call/       |
| Gmail API quota and label naming restrictions may apply; monitor usage and adapt limits accordingly.            | N/A                                                                                                |

---

**Disclaimer:**  
The provided content is generated from an automated n8n workflow. It respects all applicable content policies and handles only legal and public data.