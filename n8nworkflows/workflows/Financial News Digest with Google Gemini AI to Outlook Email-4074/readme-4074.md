Financial News Digest with Google Gemini AI to Outlook Email

https://n8nworkflows.xyz/workflows/financial-news-digest-with-google-gemini-ai-to-outlook-email-4074


# Financial News Digest with Google Gemini AI to Outlook Email

---

### 1. Workflow Overview

This workflow automates the collection, AI-powered summarization, and email distribution of financial news content. It targets users who want daily, concise, and well-structured financial news digests delivered automatically via Outlook email.

The workflow is logically divided into these blocks:

- **1.1 Input Reception and Scheduling:** Triggers the workflow either manually or on a schedule.
- **1.2 Web Scraping and Data Extraction:** Fetches financial news from specified online sources and extracts relevant HTML content.
- **1.3 Preprocessing and Looping:** Cleans and structures the scraped data, then loops through individual items for processing.
- **1.4 AI Summarization:** Utilizes Google Gemini via LangChain agents to synthesize and format news items into bullet-pointed summaries.
- **1.5 Aggregation and Output Formatting:** Aggregates the AI-generated summaries and prepares styled HTML content.
- **1.6 Email Distribution:** Sends the final formatted summary via Microsoft Outlook automatically.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Scheduling

**Overview:**  
Provides two triggers for the workflow: manual activation for testing and scheduled activation for automated daily runs.

**Nodes Involved:**  
- When clicking ‘Test workflow’ (Manual Trigger)  
- Schedule Trigger

**Node Details:**

- **When clicking ‘Test workflow’**  
  - *Type:* Manual Trigger  
  - *Role:* Allows manual start of the workflow for testing or immediate execution.  
  - *Config:* Default settings, no parameters required.  
  - *Connections:* Outputs to "Edit Fields1".  
  - *Edge cases:* None significant; manual trigger relies on user action.

- **Schedule Trigger**  
  - *Type:* Schedule Trigger  
  - *Role:* Automatically triggers workflow based on a predefined schedule (e.g., daily).  
  - *Config:* Default cron or interval-based schedule (not explicitly shown).  
  - *Connections:* Outputs to "Edit Fields1".  
  - *Edge cases:* Scheduling misconfiguration or server downtime could prevent trigger firing.

---

#### 2.2 Web Scraping and Data Extraction

**Overview:**  
Downloads HTML content from a financial news website (Financial Times) and extracts relevant elements for further processing.

**Nodes Involved:**  
- Get financial news online (HTTP Request)  
- Gather the elements (Set)  
- Code  
- Loop Over Items (Split In Batches)  
- Edit Fields1 (Set)

**Node Details:**

- **Get financial news online**  
  - *Type:* HTTP Request  
  - *Role:* Fetches the raw HTML content from https://www.ft.com/  
  - *Config:* Default GET request to the FT homepage URL.  
  - *Connections:* Outputs to "Gather the elements".  
  - *Edge cases:* Site blocking, HTTP errors, or changed HTML structure may cause failures or incorrect data.

- **Gather the elements**  
  - *Type:* Set  
  - *Role:* Prepares or filters the fetched data to isolate relevant HTML snippets.  
  - *Config:* Sets or filters fields, possibly extracting elements using expressions or regex (exact logic not visible).  
  - *Connections:* Outputs to "Edit Fields".  
  - *Edge cases:* If data format changes or extraction expressions fail, empty or malformed data may propagate.

- **Edit Fields1**  
  - *Type:* Set  
  - *Role:* Prepares data for the "Code" node, possibly formatting or setting variables.  
  - *Connections:* Outputs to "Code".  

- **Code**  
  - *Type:* Code  
  - *Role:* Runs custom JavaScript to clean, parse, or transform the scraped HTML data into structured items suitable for AI processing.  
  - *Config:* Custom JS logic for dynamic input management and cleanup.  
  - *Connections:* Outputs to "Loop Over Items".  
  - *Edge cases:* JavaScript errors, unexpected HTML structure, or empty inputs may cause failures.

- **Loop Over Items**  
  - *Type:* Split In Batches  
  - *Role:* Processes each news item individually in batches to control load and pacing.  
  - *Config:* Batch size and concurrency settings (not explicitly shown).  
  - *Connections:* Outputs to "Aggregate" (for processed items) and back to "Get financial news online" (looping).  
  - *Edge cases:* Batch size too large may cause timeouts; looping logic must prevent infinite loops.

---

#### 2.3 AI Summarization

**Overview:**  
Uses LangChain agents connected to Google Gemini AI models to summarize and structure individual financial news items into a clear, bullet-pointed format.

**Nodes Involved:**  
- Edit Fields (Set)  
- AI Agent  
- Google Gemini Chat Model  
- Item List Output Parser  
- Edit Fields2 (Set)  
- AI Agent1  
- Google Gemini Chat Model1

**Node Details:**

- **Edit Fields**  
  - *Type:* Set  
  - *Role:* Prepares input fields and parameters for the AI Agent node.  
  - *Connections:* Outputs to "AI Agent".  

- **AI Agent**  
  - *Type:* LangChain Agent  
  - *Role:* Orchestrates the AI prompt logic and formatting using Google Gemini as the language model.  
  - *Config:* Connected to "Google Gemini Chat Model" as its language model.  
  - *Connections:* Outputs to "Edit Fields2".  
  - *Edge cases:* AI API rate limits, authentication errors, prompt failures.

- **Google Gemini Chat Model**  
  - *Type:* LangChain Google Gemini Chat Model  
  - *Role:* Provides NLP capabilities for summarization via Google Gemini API.  
  - *Connections:* Used as the language model by "AI Agent".  
  - *Edge cases:* API downtime, credential issues.

- **Item List Output Parser**  
  - *Type:* LangChain Output Parser (Item List)  
  - *Role:* Parses AI agent outputs into structured lists for further processing.  
  - *Connections:* Parses output of "AI Agent".  

- **Edit Fields2**  
  - *Type:* Set  
  - *Role:* Adjusts or cleans AI output before feeding into the second AI agent ("AI Agent1").  
  - *Connections:* Outputs to "AI Agent1".  

- **AI Agent1**  
  - *Type:* LangChain Agent  
  - *Role:* Runs a second round of AI processing, possibly refining or reformatting summaries.  
  - *Config:* Uses "Google Gemini Chat Model1" as its language model.  
  - *Connections:* Outputs back to "Loop Over Items" for batch control.  
  - *Edge cases:* Same AI API issues as above.

- **Google Gemini Chat Model1**  
  - *Type:* LangChain Google Gemini Chat Model  
  - *Role:* Language model for the second AI agent.  
  - *Connections:* Connected to "AI Agent1".  

---

#### 2.4 Aggregation and Output Formatting

**Overview:**  
Collects all processed summary items into a single document and formats it for email delivery with styled HTML and coral-colored headings.

**Nodes Involved:**  
- Aggregate  
- Send the summary by e-mail

**Node Details:**

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Combines all individual AI-generated summaries into one consolidated output.  
  - *Config:* Default or custom aggregation logic to concatenate or merge text blocks.  
  - *Connections:* Outputs to "Send the summary by e-mail".  
  - *Edge cases:* Empty inputs or aggregation failures could cause no email content.

---

#### 2.5 Email Distribution

**Overview:**  
Sends the final formatted financial news summary as an email via Microsoft Outlook.

**Nodes Involved:**  
- Send the summary by e-mail (Microsoft Outlook)

**Node Details:**

- **Send the summary by e-mail**  
  - *Type:* Microsoft Outlook  
  - *Role:* Sends out the HTML-formatted summary via Outlook email.  
  - *Config:* Uses Outlook OAuth2 credentials. Email fields (recipient, subject, body) configured with aggregated content.  
  - *Connections:* Terminal node; no outputs.  
  - *Edge cases:* Credential expiry, API limits, network issues, or malformed email content.

---

### 3. Summary Table

| Node Name                 | Node Type                              | Functional Role                                      | Input Node(s)             | Output Node(s)                  | Sticky Note                                                                                      |
|---------------------------|--------------------------------------|-----------------------------------------------------|---------------------------|--------------------------------|-------------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger                       | Manual start trigger                                |                           | Edit Fields1                   |                                                                                                 |
| Schedule Trigger           | Schedule Trigger                      | Scheduled start trigger                             |                           | Edit Fields1                   |                                                                                                 |
| Edit Fields1              | Set                                  | Prepares data for parsing and processing           | When clicking ‘Test workflow’, Schedule Trigger | Code                          |                                                                                                 |
| Get financial news online  | HTTP Request                         | Retrieves raw HTML content from FT website          | Loop Over Items            | Gather the elements            | Url : https://www.ft.com/                                                                        |
| Gather the elements        | Set                                  | Extracts relevant HTML elements                      | Get financial news online  | Edit Fields                   |                                                                                                 |
| Code                      | Code                                 | Parses and cleans scraped HTML                       | Edit Fields1               | Loop Over Items                |                                                                                                 |
| Loop Over Items            | Split In Batches                     | Processes items batch-wise and controls looping     | Code, AI Agent1            | Aggregate, Get financial news online |                                                                                                 |
| Edit Fields               | Set                                  | Prepares AI prompt inputs                            | Gather the elements        | AI Agent                     |                                                                                                 |
| AI Agent                  | LangChain Agent                      | Summarizes news items using Google Gemini AI        | Edit Fields                | Edit Fields2                  |                                                                                                 |
| Google Gemini Chat Model   | LangChain Google Gemini Model        | Provides language model for AI Agent                 |                           | AI Agent                     |                                                                                                 |
| Item List Output Parser    | LangChain Output Parser (Item List)  | Parses AI output into structured lists               | AI Agent                   | AI Agent                     |                                                                                                 |
| Edit Fields2              | Set                                  | Prepares refined data for second AI processing       | AI Agent                   | AI Agent1                    |                                                                                                 |
| AI Agent1                 | LangChain Agent                      | Second AI summarization/refinement                    | Edit Fields2               | Loop Over Items               |                                                                                                 |
| Google Gemini Chat Model1  | LangChain Google Gemini Model        | Language model for second AI Agent                    |                           | AI Agent1                    |                                                                                                 |
| Aggregate                 | Aggregate                           | Combines all summaries into one output                | Loop Over Items            | Send the summary by e-mail    |                                                                                                 |
| Send the summary by e-mail | Microsoft Outlook                   | Sends HTML summary email to recipients                | Aggregate                 |                                |                                                                                                 |
| Sticky Note               | Sticky Note                         | [Empty]                                              |                           |                                |                                                                                                 |
| Sticky Note1              | Sticky Note                         | [Empty]                                              |                           |                                |                                                                                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Manual Trigger node:**  
   - Type: Manual Trigger  
   - Purpose: Allows manual execution for testing.

2. **Create a Schedule Trigger node:**  
   - Type: Schedule Trigger  
   - Configure schedule (e.g., daily at a set time).

3. **Create a Set node ("Edit Fields1"):**  
   - Prepare any initial fields or parameters to start processing.  
   - Connect outputs from both Manual Trigger and Schedule Trigger to this node.

4. **Create a Code node:**  
   - Write JavaScript to parse and clean HTML content.  
   - Input: Data prepared by "Edit Fields1".  
   - Output: Structured array of news items.

5. **Create an HTTP Request node ("Get financial news online"):**  
   - Set method to GET.  
   - URL: https://www.ft.com/  
   - No authentication required.  
   - Connect output to a Set node.

6. **Create a Set node ("Gather the elements"):**  
   - Extract or isolate relevant HTML elements from the HTTP response.  
   - Use expressions or JSON path to filter content.

7. **Connect "Gather the elements" to "Edit Fields" Set node:**  
   - Prepare fields for AI processing.

8. **Create a Split In Batches node ("Loop Over Items"):**  
   - Configure batch size to process news items individually or in small sets.  
   - Connect output from "Code" node to this node.

9. **Create a LangChain Google Gemini Chat Model node ("Google Gemini Chat Model"):**  
   - Set up credentials for Google Gemini API.  
   - Use as language model for AI agent.

10. **Create a LangChain Agent node ("AI Agent"):**  
    - Set the language model to "Google Gemini Chat Model".  
    - Configure prompt logic for financial news summarization.  
    - Connect input from "Edit Fields" Set node.

11. **Create a LangChain Output Parser node ("Item List Output Parser"):**  
    - Parses AI output into structured lists.  
    - Connect output from "AI Agent".

12. **Create a Set node ("Edit Fields2"):**  
    - Prepare or clean AI output for further processing.

13. **Create a second LangChain Google Gemini Chat Model node ("Google Gemini Chat Model1"):**  
    - Duplicate setup for Google Gemini credentials.

14. **Create a second LangChain Agent node ("AI Agent1"):**  
    - Use "Google Gemini Chat Model1" as language model.  
    - Connect input from "Edit Fields2".  
    - Connect output back to "Loop Over Items" to control batch flow.

15. **Create an Aggregate node:**  
    - Aggregate all processed summaries into one output.

16. **Create a Microsoft Outlook node ("Send the summary by e-mail"):**  
    - Configure Outlook OAuth2 credentials.  
    - Set recipient, subject, and HTML body using aggregated content.  
    - Connect input from Aggregate node.

17. **Connect nodes according to logical flow:**  
    - Manual Trigger & Schedule Trigger → Edit Fields1 → Code → Loop Over Items  
    - Loop Over Items (batch) → Get financial news online → Gather the elements → Edit Fields → AI Agent → Edit Fields2 → AI Agent1 → Loop Over Items  
    - Loop Over Items → Aggregate → Send the summary by e-mail

18. **Configure credentials:**  
    - Google Gemini API credentials for LangChain nodes.  
    - Microsoft Outlook OAuth2 credentials for email node.

19. **Test entire workflow:**  
    - Trigger manually and verify each step.

---

### 5. General Notes & Resources

| Note Content                                                                                             | Context or Link                                |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------|
| The workflow uses Google Gemini via LangChain agents for advanced AI summarization of financial news.  | AI NLP technology integration                   |
| Outlook node requires OAuth2 credentials properly configured to send emails automatically.              | Microsoft Outlook API integration               |
| The FT website scraping may not work if the website changes structure or blocks requests.               | Web scraping reliability note                    |
| Custom JavaScript code is used to parse and clean messy HTML for better AI input.                       | Custom JS logic handling                          |
| Coral-colored headings in email output enhance readability and branding consistency.                    | Email styling design choice                      |
| For more information on n8n LangChain integration, see https://docs.n8n.io/integrations/ai/langchain/   | Official n8n LangChain docs                       |

---

**Disclaimer:** The text provided is derived exclusively from an automated workflow created with n8n, an integration and automation tool. This process strictly adheres to current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly accessible.