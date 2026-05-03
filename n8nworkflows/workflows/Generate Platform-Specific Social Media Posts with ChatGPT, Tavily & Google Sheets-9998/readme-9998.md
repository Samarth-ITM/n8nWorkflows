Generate Platform-Specific Social Media Posts with ChatGPT, Tavily & Google Sheets

https://n8nworkflows.xyz/workflows/generate-platform-specific-social-media-posts-with-chatgpt--tavily---google-sheets-9998


# Generate Platform-Specific Social Media Posts with ChatGPT, Tavily & Google Sheets

---

### 1. Workflow Overview

This workflow automates the generation of platform-specific social media posts by leveraging ChatGPT, Tavily's AI search capabilities, and Google Sheets for data input/output. Its primary goal is to create tailored content for LinkedIn, X (Twitter), and Instagram based on campaign data, enabling efficient multi-platform social media management.

The workflow is logically structured into the following blocks:

- **1.1 Input Reception and Preparation:** Triggered by changes in a Google Sheet, it sets up search parameters for content generation.
- **1.2 AI Search and Data Splitting:** Uses Tavily AI search node to retrieve relevant data, then splits and aggregates it for further processing.
- **1.3 AI Content Generation:** Uses ChatGPT models configured specifically for LinkedIn, X, and Instagram to generate platform-tailored posts.
- **1.4 Update Campaign Output:** Writes the generated content back into the Google Sheet to update campaign data.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preparation

- **Overview:**  
This block initiates the workflow by detecting changes in a Google Sheet, then prepares the required search parameters to query Tavily’s AI search.

- **Nodes Involved:**  
  - Google Sheets Trigger  
  - Set Search Fields

- **Node Details:**

  - **Google Sheets Trigger**  
    - *Type & Role:* Event trigger node that activates the workflow on changes in a specific Google Sheet.  
    - *Configuration:* Watches for new or updated rows in the spreadsheet.  
    - *Expressions/Variables:* Typically uses sheet ID, worksheet name, and trigger conditions.  
    - *Connections:* Outputs to "Set Search Fields".  
    - *Version:* Compatible with n8n v1.   
    - *Edge Cases:* Possible failure if sheet permissions change or API limits are exceeded.

  - **Set Search Fields**  
    - *Type & Role:* Sets or transforms input data from the sheet into parameters suitable for the Tavily search node.  
    - *Configuration:* Static or dynamic assignment of search query fields, possibly including campaign keywords or topics.  
    - *Expressions:* Likely uses expressions to extract values from the Google Sheets Trigger data.  
    - *Connections:* Outputs to "Search".  
    - *Edge Cases:* Expression errors if expected fields are missing in the sheet data.

#### 2.2 AI Search and Data Splitting

- **Overview:**  
This block queries Tavily's AI search node using prepared parameters, then splits and aggregates the returned data to structure it for multiple social media platforms.

- **Nodes Involved:**  
  - Search (Tavily)  
  - Split Out  
  - Aggregate

- **Node Details:**

  - **Search (Tavily)**  
    - *Type & Role:* Calls Tavily AI search to retrieve relevant content snippets or ideas based on search fields.  
    - *Configuration:* Uses credentials for Tavily; parameters set from the previous node.  
    - *Expressions:* Inputs derived from "Set Search Fields".  
    - *Connections:* Outputs to "Split Out".  
    - *Edge Cases:* API auth failures, no results found, or rate limiting.

  - **Split Out**  
    - *Type & Role:* Splits the array of results into individual items for parallel processing.  
    - *Configuration:* Default splitting by array elements.  
    - *Connections:* Outputs to "Aggregate".  
    - *Edge Cases:* Empty inputs cause no output; malformed data can cause failures.

  - **Aggregate**  
    - *Type & Role:* Aggregates or restructures split data, possibly grouping or formatting it for AI content generation.  
    - *Configuration:* Aggregation with default or customized settings (e.g., grouping by platform).  
    - *Connections:* Outputs to "LinkedIn".  
    - *Edge Cases:* Aggregation failures if input data is inconsistent.

#### 2.3 AI Content Generation

- **Overview:**  
This block generates tailored social media post content for each platform (LinkedIn, X, Instagram) using dedicated ChatGPT language models via LangChain integration.

- **Nodes Involved:**  
  - LinkedIn (LangChain Agent)  
  - X (LangChain Agent)  
  - IG (LangChain Agent)  
  - ChatGPT Model for LinkedIn  
  - ChatGPT Model for X (Twitter)  
  - ChatGPT Model for Instagram

- **Node Details:**

  - **ChatGPT Model for LinkedIn**  
    - *Type & Role:* AI language model node configured with OpenAI credentials to generate LinkedIn-specific text.  
    - *Configuration:* Model parameters tuned for LinkedIn style and length.  
    - *Connections:* Provides language model input to "LinkedIn" agent node.  
    - *Edge Cases:* API quota limits, network timeouts.

  - **LinkedIn (LangChain Agent)**  
    - *Type & Role:* Agent node that orchestrates prompt and response processing for LinkedIn content generation.  
    - *Configuration:* Receives prompts and calls language model node; processes AI outputs.  
    - *Connections:* Outputs to "X" node.  
    - *Edge Cases:* Expression failures, invalid AI responses.

  - **ChatGPT Model for X (Twitter)**  
    - *Type & Role:* OpenAI model node tailored for X (Twitter) content.  
    - *Connections:* Inputs to "X" agent node.  
    - *Edge Cases:* Similar to LinkedIn model.

  - **X (LangChain Agent)**  
    - *Type & Role:* Processes prompts and AI responses for X platform content.  
    - *Connections:* Outputs to "IG".  
    - *Edge Cases:* As above.

  - **ChatGPT Model for Instagram**  
    - *Type & Role:* OpenAI model node for Instagram content generation.  
    - *Connections:* Inputs to "IG" agent node.  
    - *Edge Cases:* As above.

  - **IG (LangChain Agent)**  
    - *Type & Role:* Generates Instagram post content by processing AI outputs.  
    - *Connections:* Outputs to "Update Campaign".  
    - *Edge Cases:* As above.

#### 2.4 Update Campaign Output

- **Overview:**  
Writes the generated social media posts back into the Google Sheet to update the campaign data with the newly created content.

- **Nodes Involved:**  
  - Update Campaign (Google Sheets)

- **Node Details:**

  - **Update Campaign**  
    - *Type & Role:* Google Sheets node that updates rows with generated content for each platform.  
    - *Configuration:* Uses sheet ID, worksheet name, and row identifiers to update specific cells.  
    - *Connections:* Final node in the workflow.  
    - *Edge Cases:* Write permission errors, sheet locked or unavailable, API rate limits.

---

### 3. Summary Table

| Node Name                  | Node Type                        | Functional Role                           | Input Node(s)             | Output Node(s)           | Sticky Note                      |
|----------------------------|---------------------------------|-----------------------------------------|---------------------------|--------------------------|---------------------------------|
| Google Sheets Trigger       | Google Sheets Trigger            | Triggers workflow on sheet change       |                           | Set Search Fields         |                                 |
| Set Search Fields           | Set                             | Prepares search parameters               | Google Sheets Trigger      | Search                   |                                 |
| Search                     | Tavily AI Search                 | Retrieves relevant content data          | Set Search Fields          | Split Out                |                                 |
| Split Out                  | Split Out                       | Splits array of search results           | Search                    | Aggregate                |                                 |
| Aggregate                  | Aggregate                       | Aggregates split search results          | Split Out                 | LinkedIn                 |                                 |
| LinkedIn                   | LangChain Agent                 | Generates LinkedIn post content           | Aggregate                 | X                        |                                 |
| ChatGPT Model for LinkedIn | LangChain LM ChatOpenAI         | OpenAI model configured for LinkedIn    |                           | LinkedIn (ai_languageModel) |                                 |
| X                         | LangChain Agent                 | Generates X (Twitter) post content        | LinkedIn                  | IG                       |                                 |
| ChatGPT Model for X (Twitter) | LangChain LM ChatOpenAI         | OpenAI model configured for X            |                           | X (ai_languageModel)      |                                 |
| IG                        | LangChain Agent                 | Generates Instagram post content          | X                         | Update Campaign           |                                 |
| ChatGPT Model for Instagram | LangChain LM ChatOpenAI         | OpenAI model configured for Instagram    |                           | IG (ai_languageModel)     |                                 |
| Update Campaign            | Google Sheets                   | Updates campaign data with generated content | IG                      |                          |                                 |
| Sticky Note                | Sticky Note                    | (Empty content)                           |                           |                          |                                 |
| Sticky Note1               | Sticky Note                    | (Empty content)                           |                           |                          |                                 |
| Sticky Note2               | Sticky Note                    | (Empty content)                           |                           |                          |                                 |
| Sticky Note3               | Sticky Note                    | (Empty content)                           |                           |                          |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Google Sheets Trigger Node**  
   - Type: Google Sheets Trigger  
   - Configure to watch the specific sheet and worksheet for new or updated rows.  
   - Set appropriate polling or webhook method.  

2. **Add Set Node (Set Search Fields)**  
   - Type: Set  
   - Extract and set the search query fields from the Google Sheets Trigger data.  
   - Map relevant fields such as campaign keywords or topics.  
   - Connect output from Google Sheets Trigger to this node.  

3. **Add Tavily Search Node**  
   - Type: Tavily AI Search  
   - Configure with Tavily API credentials.  
   - Set search parameters using data from "Set Search Fields".  
   - Connect output from "Set Search Fields" to this node.  

4. **Add Split Out Node**  
   - Type: Split Out  
   - Configure to split the array of results from Tavily Search into individual items.  
   - Connect output from "Search" node to this node.  

5. **Add Aggregate Node**  
   - Type: Aggregate  
   - Configure to aggregate or restructure the split data as needed (e.g., grouping by platform).  
   - Connect output from "Split Out" to this node.  

6. **Add LangChain Agent Node for LinkedIn**  
   - Type: LangChain Agent  
   - Configure prompt and settings for LinkedIn post style.  
   - Connect output from "Aggregate" to this node.  

7. **Add ChatGPT Model Node for LinkedIn**  
   - Type: LangChain LM ChatOpenAi  
   - Configure with OpenAI credentials.  
   - Set model parameters optimized for LinkedIn tone and length.  
   - Connect AI language model output to "LinkedIn" agent node.  

8. **Add LangChain Agent Node for X (Twitter)**  
   - Type: LangChain Agent  
   - Configure prompt for X (Twitter) style posts.  
   - Connect output from "LinkedIn" agent node to this node.  

9. **Add ChatGPT Model Node for X (Twitter)**  
   - Type: LangChain LM ChatOpenAi  
   - Configure with OpenAI credentials.  
   - Set model parameters optimized for Twitter brevity and style.  
   - Connect AI language model output to "X" agent node.  

10. **Add LangChain Agent Node for Instagram**  
    - Type: LangChain Agent  
    - Configure prompt for Instagram content style.  
    - Connect output from "X" agent node to this node.  

11. **Add ChatGPT Model Node for Instagram**  
    - Type: LangChain LM ChatOpenAi  
    - Configure with OpenAI credentials.  
    - Set model parameters optimized for Instagram captions.  
    - Connect AI language model output to "IG" agent node.  

12. **Add Google Sheets Node (Update Campaign)**  
    - Type: Google Sheets  
    - Configure to update the same sheet and worksheet with generated social media posts.  
    - Map fields to appropriate columns for LinkedIn, X, and Instagram posts.  
    - Connect output from "IG" agent node to this node.  

13. **Configure Credentials**  
    - Setup Google Sheets credentials with read/write permissions.  
    - Setup OpenAI credentials for ChatGPT model nodes.  
    - Setup Tavily API credentials for the search node.  

14. **Test Workflow**  
    - Trigger a sheet update or add a new row to test end-to-end functionality.  
    - Monitor logs for errors such as API limits or expression failures.

---

### 5. General Notes & Resources

| Note Content                                                                                  | Context or Link                                 |
|-----------------------------------------------------------------------------------------------|------------------------------------------------|
| Workflow designed to generate platform-specific social media posts automatically.             | General description                             |
| Uses LangChain Agent nodes to orchestrate ChatGPT calls tailored per social media platform.   | https://n8n.io/integrations/langchain          |
| Tavily node offers semantic AI search capabilities to enrich content generation prompts.     | https://tavily.ai                               |
| Google Sheets used as both input trigger and output destination for campaign data management. | https://developers.google.com/sheets/api       |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---