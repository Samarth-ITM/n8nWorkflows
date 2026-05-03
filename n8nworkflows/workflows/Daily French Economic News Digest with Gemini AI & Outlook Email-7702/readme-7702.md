Daily French Economic News Digest with Gemini AI & Outlook Email

https://n8nworkflows.xyz/workflows/daily-french-economic-news-digest-with-gemini-ai---outlook-email-7702


# Daily French Economic News Digest with Gemini AI & Outlook Email

### 1. Workflow Overview

This workflow automates the process of gathering daily French economic news from multiple RSS feeds, consolidating and filtering the articles, generating a summarized digest using Google Gemini AI, and sending the summary via Microsoft Outlook email. It is designed for users interested in receiving a curated and concise update on French economic news every day.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Scheduled trigger initiates the workflow daily; multiple RSS feed reads gather news articles.
- **1.2 Data Filtering and Sorting:** Conditional filtering nodes check for relevant content; sorting and limiting nodes organize and reduce items.
- **1.3 Data Merging:** Merges news items from multiple RSS sources into a unified list.
- **1.4 AI Processing:** Sends merged news to a Google Gemini AI model for summarization and structured output parsing.
- **1.5 Post-Processing and Aggregation:** Adjusts fields and aggregates processed data.
- **1.6 Output Dispatch:** Sends the generated summary via Outlook email.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block triggers the workflow daily and reads news articles from three distinct RSS feeds.
- **Nodes Involved:** Schedule Trigger, RSS Read, RSS Read1, RSS Read2
- **Node Details:**

  - **Schedule Trigger**  
    - *Type:* Trigger  
    - *Role:* Initiates the workflow, likely set to run daily (exact schedule not configured in JSON).  
    - *Connections:* Outputs to RSS Read, RSS Read1, RSS Read2 simultaneously.  
    - *Edge cases:* Trigger misconfiguration could prevent workflow start.

  - **RSS Read, RSS Read1, RSS Read2**  
    - *Type:* RSS Feed Reader  
    - *Role:* Fetches news articles from specified RSS feed URLs (not provided in JSON; must be configured).  
    - *Parameters:* Empty in JSON, requires feed URLs to be set.  
    - *Input:* From Schedule Trigger.  
    - *Output:* To respective If nodes (If, If1, If2).  
    - *Edge cases:* Network failures, invalid or empty feeds, malformed RSS XML.

---

#### 1.2 Data Filtering and Sorting

- **Overview:** Filters and sorts the news items from each RSS feed to ensure relevance and manage the data volume.
- **Nodes Involved:** If, If1, If2, Sort, Sort1, Sort2, Limit, Limit1, Limit2
- **Node Details:**

  - **If, If1, If2**  
    - *Type:* Conditional  
    - *Role:* Filter news items based on criteria (criteria not specified, likely on article content or metadata).  
    - *Input:* From RSS Read nodes.  
    - *Output:* True branch to corresponding Sort nodes.  
    - *Edge cases:* Incorrect conditions may filter out all items or allow irrelevant items.

  - **Sort, Sort1, Sort2**  
    - *Type:* Sorting  
    - *Role:* Sort filtered news items, presumably by date or relevance.  
    - *Input:* From If nodes.  
    - *Output:* To corresponding Limit nodes.  
    - *Edge cases:* Sorting on missing or malformed fields may cause errors.

  - **Limit, Limit1, Limit2**  
    - *Type:* Limit  
    - *Role:* Restricts the number of articles passed downstream to control payload size.  
    - *Input:* From Sort nodes.  
    - *Output:* To Merge node (Limit, Limit1, Limit2 each connected to different inputs).  
    - *Edge cases:* Limit too low may omit important data; too high may overload AI processing.

---

#### 1.3 Data Merging

- **Overview:** Combines the limited news items from all three feeds into a single dataset for unified processing.
- **Nodes Involved:** Merge
- **Node Details:**

  - **Merge**  
    - *Type:* Merge  
    - *Role:* Combines multiple input streams (three from Limit nodes) into one output stream.  
    - *Input:* From Limit, Limit1, Limit2 (inputs indexed 0,1,2 respectively).  
    - *Output:* To Rédaction (AI Processing) node.  
    - *Edge cases:* Mismatched data structures can cause merge failures.

---

#### 1.4 AI Processing

- **Overview:** Uses Google Gemini AI models to generate a summarized digest of the merged news articles and parse the structured output.
- **Nodes Involved:** Rédaction, Google Gemini Chat Model2, Google Gemini Chat Model3, Structured Output Parser1
- **Node Details:**

  - **Google Gemini Chat Model2**  
    - *Type:* Google Gemini Language Model Chat  
    - *Role:* Provides AI language model capabilities for the Rédaction node.  
    - *Input:* Connected as AI language model resource for Rédaction.  
    - *Edge cases:* API key or quota issues, model downtime.

  - **Rédaction**  
    - *Type:* Langchain Agent  
    - *Role:* Orchestrates the AI prompt to generate a news summary.  
    - *Input:* From Merge node (news articles).  
    - *Output:* To Edit Fields3 node.  
    - *AI Language Model Input:* Uses Google Gemini Chat Model2.  
    - *AI Output Parser:* Connected to Structured Output Parser1 for structured results.  
    - *Edge cases:* Prompt failures, malformed input data, AI response delays.

  - **Google Gemini Chat Model3**  
    - *Type:* Google Gemini Language Model Chat  
    - *Role:* Provides language model for Structured Output Parser1.  
    - *Input:* AI language model connection from Structured Output Parser1.  
    - *Edge cases:* Same as Google Gemini Chat Model2.

  - **Structured Output Parser1**  
    - *Type:* Structured Output Parser  
    - *Role:* Parses the AI's output into structured JSON or other formats for further use.  
    - *Input:* AI output from Rédaction.  
    - *Output:* To Rédaction node (feedback loop for parsing).  
    - *Edge cases:* Parsing errors if AI output format differs from expected.

---

#### 1.5 Post-Processing and Aggregation

- **Overview:** Adjusts fields for the final data structure and aggregates the summary content before sending.
- **Nodes Involved:** Edit Fields3, Aggregate1
- **Node Details:**

  - **Edit Fields3**  
    - *Type:* Set  
    - *Role:* Modifies or adds fields to the AI-generated summary to prepare for aggregation or email.  
    - *Input:* From Rédaction.  
    - *Output:* To Aggregate1.  
    - *Edge cases:* Misconfiguration may remove or corrupt important fields.

  - **Aggregate1**  
    - *Type:* Aggregate  
    - *Role:* Aggregates processed data items into a single output (e.g., concatenating summaries).  
    - *Input:* From Edit Fields3.  
    - *Output:* To Send the summary by e-mail1.  
    - *Edge cases:* Large payloads may cause timeouts.

---

#### 1.6 Output Dispatch

- **Overview:** Sends the aggregated summary as an email via Microsoft Outlook.
- **Nodes Involved:** Send the summary by e-mail1
- **Node Details:**

  - **Send the summary by e-mail1**  
    - *Type:* Microsoft Outlook  
    - *Role:* Sends an email with the daily economic news digest summary.  
    - *Input:* From Aggregate1.  
    - *Parameters:* Requires Outlook credentials and email content details (not shown in JSON).  
    - *Edge cases:* Authentication failures, SMTP errors, invalid email addresses.  
    - *Webhook ID present:* Possibly configured to handle webhook responses or triggers.

---

### 3. Summary Table

| Node Name                   | Node Type                             | Functional Role                          | Input Node(s)                         | Output Node(s)                      | Sticky Note |
|-----------------------------|-------------------------------------|----------------------------------------|-------------------------------------|-----------------------------------|-------------|
| Schedule Trigger            | Schedule Trigger                    | Initiates workflow daily                | -                                   | RSS Read, RSS Read1, RSS Read2    |             |
| RSS Read                   | RSS Feed Read                      | Reads first RSS feed                    | Schedule Trigger                    | If                                |             |
| RSS Read1                  | RSS Feed Read                      | Reads second RSS feed                   | Schedule Trigger                    | If1                               |             |
| RSS Read2                  | RSS Feed Read                      | Reads third RSS feed                    | Schedule Trigger                    | If2                               |             |
| If                         | If                                | Filters first feed results              | RSS Read                           | Sort                              |             |
| If1                        | If                                | Filters second feed results             | RSS Read1                          | Sort1                             |             |
| If2                        | If                                | Filters third feed results              | RSS Read2                          | Sort2                             |             |
| Sort                       | Sort                              | Sorts first feed filtered items         | If                                | Limit                             |             |
| Sort1                      | Sort                              | Sorts second feed filtered items        | If1                               | Limit1                            |             |
| Sort2                      | Sort                              | Sorts third feed filtered items         | If2                               | Limit2                            |             |
| Limit                      | Limit                             | Limits first feed sorted items          | Sort                              | Merge (input 0)                   |             |
| Limit1                     | Limit                             | Limits second feed sorted items         | Sort1                             | Merge (input 1)                   |             |
| Limit2                     | Limit                             | Limits third feed sorted items          | Sort2                             | Merge (input 2)                   |             |
| Merge                      | Merge                             | Merges all limited feed items           | Limit, Limit1, Limit2              | Rédaction                        |             |
| Rédaction                  | Langchain Agent                   | AI summarization of merged news         | Merge                            | Edit Fields3                     |             |
| Google Gemini Chat Model2  | Google Gemini AI Chat Model       | Provides AI language model for summary  | -                               | Rédaction (AI model input)        |             |
| Google Gemini Chat Model3  | Google Gemini AI Chat Model       | Provides AI language model for parsing  | -                               | Structured Output Parser1 (AI model input) |             |
| Structured Output Parser1  | Structured Output Parser          | Parses AI output into structured format | Rédaction (AI output)            | Rédaction (feedback for parsing)  |             |
| Edit Fields3               | Set                              | Adjusts fields for final aggregation    | Rédaction                         | Aggregate1                       |             |
| Aggregate1                 | Aggregate                        | Aggregates summaries for email          | Edit Fields3                     | Send the summary by e-mail1      |             |
| Send the summary by e-mail1| Microsoft Outlook                 | Sends summary email                      | Aggregate1                      | -                                 |             |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node**  
   - Set to trigger daily at desired time.

2. **Create Three RSS Feed Read Nodes:**  
   - Name them RSS Read, RSS Read1, RSS Read2.  
   - Configure each with the URL of a French economic news RSS feed.  
   - Connect Schedule Trigger’s output to all three nodes in parallel.

3. **Create Three If Nodes (If, If1, If2):**  
   - Connect RSS Read → If, RSS Read1 → If1, RSS Read2 → If2.  
   - Configure filtering criteria to keep only relevant articles (e.g., language = French, keywords in title or description).

4. **Create Three Sort Nodes (Sort, Sort1, Sort2):**  
   - Connect If → Sort, If1 → Sort1, If2 → Sort2.  
   - Configure to sort articles by publication date descending.

5. **Create Three Limit Nodes (Limit, Limit1, Limit2):**  
   - Connect Sort → Limit, Sort1 → Limit1, Sort2 → Limit2.  
   - Set the limit to a reasonable number (e.g., 5-10 articles per feed).

6. **Create Merge Node:**  
   - Connect Limit, Limit1, Limit2 to inputs 0, 1, and 2 of the Merge node respectively.  
   - Configure to merge input arrays (e.g., "Merge By" mode to combine all items).

7. **Create Langchain Agent Node (Rédaction):**  
   - Connect Merge’s output to Rédaction node input.  
   - Configure agent with a prompt to generate a concise French economic news summary from the input articles.  
   - Select Google Gemini Chat Model2 as the AI language model resource.

8. **Create Google Gemini Chat Model Nodes (Google Gemini Chat Model2, Google Gemini Chat Model3):**  
   - Configure credentials with Google Cloud API and select Gemini AI chat models.  
   - Connect Google Gemini Chat Model2 as AI language model input to Rédaction.  
   - Connect Google Gemini Chat Model3 as AI language model input to Structured Output Parser1.

9. **Create Structured Output Parser Node (Structured Output Parser1):**  
   - Connect AI output from Rédaction to Structured Output Parser1 input.  
   - Define expected output schema for the parsed summary.  
   - Connect parser output back to Rédaction for integrated processing.

10. **Create Set Node (Edit Fields3):**  
    - Connect Rédaction output to Edit Fields3.  
    - Configure to adjust fields for email formatting (e.g., set subject, body fields).

11. **Create Aggregate Node (Aggregate1):**  
    - Connect Edit Fields3 output to Aggregate1.  
    - Configure to aggregate all summary parts into a single output.

12. **Create Microsoft Outlook Node (Send the summary by e-mail1):**  
    - Connect Aggregate1 output to this node.  
    - Configure Outlook OAuth2 credentials.  
    - Set recipient email, subject, and email body fields from aggregated data.

13. **Test the workflow end-to-end:**  
    - Ensure RSS feed URLs are valid and accessible.  
    - Verify AI credentials and quota.  
    - Confirm Outlook credentials and email sending capability.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow designed to aggregate French economic news and generate a digest using Google Gemini AI and send via Outlook email.                                    | Workflow Purpose                                                                                 |
| Requires valid Google Cloud account with Gemini AI access and Microsoft Outlook OAuth2 credentials for email sending.                                            | Credential Setup Requirement                                                                     |
| For detailed info on Google Gemini models in n8n, see: https://docs.n8n.io/integrations/ai/google-gemini/                                                       | Documentation Link                                                                              |
| RSS feed URLs must be configured manually as they are empty in the provided JSON.                                                                                | User Configuration Required                                                                     |
| Possible improvements include adding error handling nodes after RSS Reads and AI nodes to catch and manage API or network failures gracefully.                  | Robustness Recommendation                                                                       |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and publicly available.