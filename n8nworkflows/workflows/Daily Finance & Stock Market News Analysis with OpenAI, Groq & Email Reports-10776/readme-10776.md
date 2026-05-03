Daily Finance & Stock Market News Analysis with OpenAI, Groq & Email Reports

https://n8nworkflows.xyz/workflows/daily-finance---stock-market-news-analysis-with-openai--groq---email-reports-10776


# Daily Finance & Stock Market News Analysis with OpenAI, Groq & Email Reports

---
### 1. Workflow Overview

This workflow automates the daily retrieval, classification, summarization, expert commentary addition, archiving, and reporting of finance and stock market news articles. It is designed to fetch relevant news from multiple RSS feeds each morning (8 AM), filter and classify articles related to finance and insurance using AI models, summarize the top articles, enrich them with expert commentary powered by Groq AI, and then archive and send a formatted email report.

**Target Use Cases:**  
- Financial analysts or investment managers wanting a daily digest of relevant market news.  
- Automated content curation and enrichment for financial newsletters.  
- Integration of AI summarization and expert insight into news workflows.

**Logical Blocks:**

- **1.1 Input Reception and RSS Feed Retrieval:** Scheduled trigger initiates the workflow and configures RSS feed URLs to fetch recent news articles.

- **1.2 Article Filtering and Normalization:** Filters articles published within the last 24 hours and normalizes data for downstream AI processing.

- **1.3 AI Classification:** Uses OpenAI to classify articles focused on finance and insurance.

- **1.4 Aggregation and Summarization:** Aggregates classified articles and summarizes the top 15 using OpenAI models.

- **1.5 Expert Commentary Addition:** Adds domain expert commentary to summaries using Groq AI.

- **1.6 Result Splitting, Transformation, and Archiving:** Splits detailed analysis results, transforms data for Google Sheets, archives links, and saves summaries.

- **1.7 Email Formatting and Delivery:** Formats the report for desktop (and optionally mobile) and sends it via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and RSS Feed Retrieval

- **Overview:**  
  This block triggers the workflow daily at 8 AM, sets the RSS feed URLs, splits them for individual processing, and fetches content from each RSS feed.

- **Nodes Involved:**  
  - Daily RSS Trigger (8AM)  
  - Configure RSS Sources  
  - Split RSS URLs  
  - Fetch RSS Content

- **Node Details:**  

  - **Daily RSS Trigger (8AM)**  
    - Type: Schedule Trigger  
    - Role: Initiates the workflow daily at 8 AM.  
    - Configuration: Default schedule trigger set for 8 AM daily.  
    - Inputs: None  
    - Outputs: Triggers "Configure RSS Sources".  
    - Edge Cases: Trigger may fail if n8n instance has downtime at scheduled time.

  - **Configure RSS Sources**  
    - Type: Set  
    - Role: Defines the list of RSS feed URLs to be processed.  
    - Configuration: Sets an array or list of finance-related RSS feed URLs.  
    - Inputs: From trigger node.  
    - Outputs: Feeds into "Split RSS URLs".  
    - Edge Cases: Invalid or unreachable URLs can cause fetch failures downstream.

  - **Split RSS URLs**  
    - Type: SplitOut  
    - Role: Splits the list of RSS URLs so each URL is processed individually.  
    - Configuration: Standard split on each item in the RSS URL list.  
    - Inputs: RSS URLs from "Configure RSS Sources".  
    - Outputs: One output per RSS URL to "Fetch RSS Content".  
    - Edge Cases: Empty list leads to no downstream processing.

  - **Fetch RSS Content**  
    - Type: RSS Feed Read  
    - Role: Fetches articles from each RSS feed URL.  
    - Configuration: Defaults with max 2 retries on failure, continues on error to avoid full workflow failure.  
    - Inputs: Single RSS feed URL from splitter.  
    - Outputs: Raw RSS articles to filtering stage.  
    - Edge Cases: Network errors, invalid feed format, or empty feeds.  
    - Version: 1.2  
    - Error Handling: Continues on error to allow partial success.

#### 1.2 Article Filtering and Normalization

- **Overview:**  
  Filters articles to only those published in the last 24 hours and normalizes data fields for AI consumption.

- **Nodes Involved:**  
  - Filter Recent Articles (24h)  
  - Normalize Article Data

- **Node Details:**  

  - **Filter Recent Articles (24h)**  
    - Type: Filter  
    - Role: Filters articles based on publication date within last 24 hours.  
    - Configuration: Uses expression to compare article date with current date minus 24 hours.  
    - Inputs: RSS articles from "Fetch RSS Content".  
    - Outputs: Recent articles to normalization.  
    - Edge Cases: Articles without valid date fields may be excluded.  
    - Error Handling: Continues on error.

  - **Normalize Article Data**  
    - Type: Set  
    - Role: Normalizes article data structure (e.g., title, summary, link) for AI processing.  
    - Configuration: Sets standard fields expected by the AI nodes.  
    - Inputs: Filtered articles.  
    - Outputs: Prepared articles to AI classification.  
    - Edge Cases: Missing or malformed article fields.

#### 1.3 AI Classification

- **Overview:**  
  Classifies each article to determine if it is relevant to finance/insurance topics using OpenAI.

- **Nodes Involved:**  
  - AI: Classify Finance/Insurance  
  - OpenAI Classifier Model

- **Node Details:**  

  - **AI: Classify Finance/Insurance**  
    - Type: Langchain Text Classifier  
    - Role: Orchestrates classification logic using OpenAI.  
    - Configuration: Calls the OpenAI model node for inference.  
    - Inputs: Normalized articles.  
    - Outputs: Classified articles with tags or labels.  
    - Edge Cases: API quota limits or invalid responses.  
    - Retry Policy: Disabled for failures.

  - **OpenAI Classifier Model**  
    - Type: Langchain LM Chat OpenAI  
    - Role: Executes classification prompts against OpenAI's language model.  
    - Configuration: Uses credentials for OpenAI API.  
    - Inputs: Text prompts from classifier node.  
    - Outputs: Classification results.  
    - Edge Cases: Auth failures, rate limits, or malformed prompts.  
    - Version: 1.2  

#### 1.4 Aggregation and Summarization

- **Overview:**  
  Aggregates classified articles and summarizes the top 15 articles with OpenAI.

- **Nodes Involved:**  
  - Aggregate Classified Articles  
  - AI: Summarize Top 15 Articles  
  - OpenAI Summary Model  
  - JSON Parser: Summary

- **Node Details:**  

  - **Aggregate Classified Articles**  
    - Type: Aggregate  
    - Role: Collects all classified articles into a single dataset.  
    - Configuration: Default aggregation on input items.  
    - Inputs: Classified articles.  
    - Outputs: Aggregated data to summarization block.  
    - Edge Cases: Empty input leads to empty aggregation.

  - **AI: Summarize Top 15 Articles**  
    - Type: Langchain Agent  
    - Role: Coordinates summarization using OpenAI.  
    - Configuration: Calls OpenAI summary model and parses output.  
    - Inputs: Aggregated articles.  
    - Outputs: Summary text.  
    - Retry Policy: Enabled to handle API errors.

  - **OpenAI Summary Model**  
    - Type: Langchain LM Chat OpenAI  
    - Role: Generates summarization text.  
    - Configuration: Uses OpenAI API key with appropriate model and prompt.  
    - Inputs: Summarization prompts.  
    - Outputs: Raw summary response.  
    - Edge Cases: API latency, rate limiting, or prompt errors.

  - **JSON Parser: Summary**  
    - Type: Langchain Output Parser Structured  
    - Role: Parses structured JSON output from the summarization.  
    - Inputs: Raw text from OpenAI Summary Model.  
    - Outputs: Structured summary data.  
    - Edge Cases: Parsing failures if output is malformed.  
    - Version: 1.2

#### 1.5 Expert Commentary Addition

- **Overview:**  
  Adds expert commentary to the summarized articles using Groq's AI model.

- **Nodes Involved:**  
  - Add Expert Commentary  
  - Groq Commentary Model  
  - JSON Parser: Commentary

- **Node Details:**  

  - **Add Expert Commentary**  
    - Type: Langchain Agent  
    - Role: Coordinates commentary generation calls and parsing.  
    - Inputs: Parsed summaries.  
    - Outputs: Commentary text.  
    - Configuration: Calls Groq Commentary Model and JSON parser.  
    - Edge Cases: Handling empty or malformed inputs.

  - **Groq Commentary Model**  
    - Type: Langchain LM Chat Groq  
    - Role: Generates expert commentary using Groq's language model.  
    - Inputs: Prompts from commentary agent node.  
    - Outputs: Raw commentary.  
    - Edge Cases: API errors or auth issues.  
    - Version: 1

  - **JSON Parser: Commentary**  
    - Type: Langchain Output Parser Structured  
    - Role: Parses Groq commentary structured output.  
    - Inputs: Raw commentary text.  
    - Outputs: Structured commentary data.  
    - Edge Cases: Parsing errors.  
    - Version: 1.3

#### 1.6 Result Splitting, Transformation, and Archiving

- **Overview:**  
  Splits combined analysis results for individual handling, transforms data for Google Sheets ingestion, archives all article links, and saves to Sheets.

- **Nodes Involved:**  
  - Split Analysis Results  
  - Transform for Sheets  
  - Save to Google Sheets  
  - Archive All Links

- **Node Details:**  

  - **Split Analysis Results**  
    - Type: SplitOut  
    - Role: Separates combined data into individual items for processing.  
    - Inputs: Commentary enriched summaries.  
    - Outputs: Items for sheet transformation.  
    - Edge Cases: Empty input leads to no output.

  - **Transform for Sheets**  
    - Type: Code  
    - Role: Transforms each data item into a format suitable for Google Sheets (e.g., rows/columns).  
    - Inputs: Split results.  
    - Outputs: Transformed data for Google Sheets.  
    - Edge Cases: Code execution errors, malformed data.

  - **Save to Google Sheets**  
    - Type: Google Sheets  
    - Role: Writes transformed data into a Google Sheet spreadsheet.  
    - Inputs: Transformed data.  
    - Outputs: Triggers email formatting nodes.  
    - Configuration: Requires Google OAuth2 credentials with Sheets API access.  
    - Edge Cases: API quota, auth failures, sheet access permissions.  
    - Version: 4.7

  - **Archive All Links**  
    - Type: Google Sheets  
    - Role: Archives all article links separately for record keeping.  
    - Inputs: Classified articles before aggregation.  
    - Outputs: Aggregation node (parallel branch).  
    - Edge Cases: Same as above.  
    - Version: 4.7

#### 1.7 Email Formatting and Delivery

- **Overview:**  
  Generates formatted email reports for desktop and optionally mobile clients, then sends the report via Gmail.

- **Nodes Involved:**  
  - Format Email (Desktop)  
  - Format Email (Mobile) (disabled)  
  - Send Report Email

- **Node Details:**  

  - **Format Email (Desktop)**  
    - Type: Code  
    - Role: Formats the final report email content optimized for desktop email clients.  
    - Inputs: Output from Google Sheets save node.  
    - Outputs: Email content to sending node.  
    - Edge Cases: Code errors or missing data.

  - **Format Email (Mobile)**  
    - Type: Code  
    - Role: Intended for mobile-optimized email formatting but currently disabled.  
    - Inputs: Same as desktop.  
    - Outputs: None (disabled).  
    - Edge Cases: N/A.

  - **Send Report Email**  
    - Type: Gmail  
    - Role: Sends the formatted email report to recipients.  
    - Configuration: Uses Gmail OAuth2 credentials, configured with recipient list, subject, and email body from formatting node.  
    - Inputs: Formatted email content.  
    - Edge Cases: Gmail API limits, auth failures, invalid recipient addresses.  
    - Version: 2.1

---

### 3. Summary Table

| Node Name                   | Node Type                             | Functional Role                          | Input Node(s)                   | Output Node(s)                             | Sticky Note                                                                                          |
|-----------------------------|-------------------------------------|----------------------------------------|--------------------------------|--------------------------------------------|----------------------------------------------------------------------------------------------------|
| Daily RSS Trigger (8AM)      | Schedule Trigger                    | Starts workflow daily at 8 AM           | None                           | Configure RSS Sources                      |                                                                                                    |
| Configure RSS Sources        | Set                                | Defines RSS feed URLs                   | Daily RSS Trigger (8AM)         | Split RSS URLs                            |                                                                                                    |
| Split RSS URLs               | SplitOut                           | Splits RSS feed URLs for individual fetch | Configure RSS Sources          | Fetch RSS Content                         |                                                                                                    |
| Fetch RSS Content            | RSS Feed Read                      | Fetches articles from each RSS feed     | Split RSS URLs                 | Filter Recent Articles (24h)              | Continues on fetch error; retries up to 2 times                                                    |
| Filter Recent Articles (24h) | Filter                            | Filters articles published in last 24h | Fetch RSS Content              | Normalize Article Data                    | Continues on error                                                                                  |
| Normalize Article Data       | Set                               | Normalizes article data structure       | Filter Recent Articles (24h)   | AI: Classify Finance/Insurance            |                                                                                                    |
| AI: Classify Finance/Insurance | Langchain Text Classifier         | Classifies articles as finance/insurance related | Normalize Article Data         | Aggregate Classified Articles, Archive All Links |                                                                                                    |
| OpenAI Classifier Model      | Langchain LM Chat OpenAI           | OpenAI model for classification         | AI: Classify Finance/Insurance | AI: Classify Finance/Insurance            | API key required; version 1.2                                                                      |
| Aggregate Classified Articles | Aggregate                         | Aggregates classified articles           | AI: Classify Finance/Insurance | AI: Summarize Top 15 Articles             |                                                                                                    |
| AI: Summarize Top 15 Articles | Langchain Agent                   | Summarizes top 15 articles               | Aggregate Classified Articles  | Add Expert Commentary                      | Retry enabled                                                                                      |
| OpenAI Summary Model         | Langchain LM Chat OpenAI           | OpenAI model for summarization           | AI: Summarize Top 15 Articles  | AI: Summarize Top 15 Articles, JSON Parser: Summary | API key required; version 1.3                                                                      |
| JSON Parser: Summary         | Langchain Output Parser Structured | Parses JSON summary output                | OpenAI Summary Model           | AI: Summarize Top 15 Articles             | Version 1.2                                                                                       |
| Add Expert Commentary        | Langchain Agent                   | Adds expert commentary                    | AI: Summarize Top 15 Articles, JSON Parser: Commentary | Split Analysis Results                   |                                                                                                    |
| Groq Commentary Model        | Langchain LM Chat Groq             | Groq AI model for commentary              | Add Expert Commentary          | Add Expert Commentary, JSON Parser: Commentary | API key required; version 1                                                                       |
| JSON Parser: Commentary      | Langchain Output Parser Structured | Parses Groq commentary output             | Groq Commentary Model          | Add Expert Commentary                      | Version 1.3                                                                                       |
| Split Analysis Results       | SplitOut                          | Splits combined analysis for processing  | Add Expert Commentary          | Transform for Sheets                       |                                                                                                    |
| Transform for Sheets         | Code                             | Prepares data for Google Sheets           | Split Analysis Results         | Save to Google Sheets                      |                                                                                                    |
| Save to Google Sheets        | Google Sheets                    | Saves data to Google Sheets               | Transform for Sheets           | Format Email (Desktop), Format Email (Mobile) | Requires Google OAuth2 credentials; version 4.7                                                   |
| Archive All Links            | Google Sheets                    | Archives all article links                 | AI: Classify Finance/Insurance | Aggregate Classified Articles              | Runs in parallel; requires Google OAuth2; version 4.7                                            |
| Format Email (Desktop)       | Code                             | Formats email content for desktop clients | Save to Google Sheets          | Send Report Email                         |                                                                                                    |
| Format Email (Mobile)        | Code (disabled)                  | Formats email content for mobile clients  | Save to Google Sheets          | None                                      | Disabled node                                                                                      |
| Send Report Email            | Gmail                            | Sends the formatted report email          | Format Email (Desktop)         | None                                      | Requires Gmail OAuth2 credentials; version 2.1                                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger Node: "Daily RSS Trigger (8AM)"**  
   - Type: Schedule Trigger  
   - Set to trigger daily at 8:00 AM.

2. **Create Set Node: "Configure RSS Sources"**  
   - Type: Set  
   - Add a field holding an array of finance-related RSS URLs (e.g., finance news sites).

3. **Connect "Daily RSS Trigger (8AM)" → "Configure RSS Sources"**

4. **Create SplitOut Node: "Split RSS URLs"**  
   - Type: SplitOut  
   - Configure to split the array of RSS URLs into individual items.

5. **Connect "Configure RSS Sources" → "Split RSS URLs"**

6. **Create RSS Feed Read Node: "Fetch RSS Content"**  
   - Type: RSS Feed Read  
   - Configure to read RSS feed URL from incoming data.  
   - Set Max Retries to 2.  
   - Set Error Handling to continue on error.

7. **Connect "Split RSS URLs" → "Fetch RSS Content"**

8. **Create Filter Node: "Filter Recent Articles (24h)"**  
   - Type: Filter  
   - Filter condition: article publication date >= current date/time minus 24 hours.  
   - Error Handling: continue on error.

9. **Connect "Fetch RSS Content" → "Filter Recent Articles (24h)"**

10. **Create Set Node: "Normalize Article Data"**  
    - Type: Set  
    - Map and normalize article fields (e.g., title, link, summary, publication date).

11. **Connect "Filter Recent Articles (24h)" → "Normalize Article Data"**

12. **Create Langchain Text Classifier Node: "AI: Classify Finance/Insurance"**  
    - Type: Langchain Text Classifier  
    - Connect an OpenAI LM Chat node for classification (see next step).  
    - Disable retries on failure.

13. **Create Langchain LM Chat OpenAI Node: "OpenAI Classifier Model"**  
    - Type: Langchain LM Chat OpenAI  
    - Configure OpenAI credentials.  
    - Setup appropriate prompt for finance/insurance classification.

14. **Connect "Normalize Article Data" → "AI: Classify Finance/Insurance"**  
    **Connect "OpenAI Classifier Model" → "AI: Classify Finance/Insurance" (as language model input)**

15. **Create Aggregate Node: "Aggregate Classified Articles"**  
    - Type: Aggregate  
    - Aggregate all classified articles into one dataset.

16. **Connect "AI: Classify Finance/Insurance" → "Aggregate Classified Articles"**

17. **Create Langchain Agent Node: "AI: Summarize Top 15 Articles"**  
    - Type: Langchain Agent  
    - Configure to call an OpenAI LM Chat node and output parser.

18. **Create Langchain LM Chat OpenAI Node: "OpenAI Summary Model"**  
    - Type: Langchain LM Chat OpenAI  
    - Configure OpenAI credentials and summarization prompt.

19. **Create Langchain Output Parser Structured Node: "JSON Parser: Summary"**  
    - Type: Langchain Output Parser Structured  
    - Configure to parse JSON structured summary output.

20. **Connect "Aggregate Classified Articles" → "AI: Summarize Top 15 Articles"**  
    **Connect "OpenAI Summary Model" → "AI: Summarize Top 15 Articles" (language model input)**  
    **Connect "JSON Parser: Summary" → "AI: Summarize Top 15 Articles" (output parser input)**

21. **Create Langchain Agent Node: "Add Expert Commentary"**  
    - Type: Langchain Agent  
    - Configure to call Groq LM Chat and output parser nodes.

22. **Create Langchain LM Chat Groq Node: "Groq Commentary Model"**  
    - Type: Langchain LM Chat Groq  
    - Configure Groq AI credentials and commentary prompt.

23. **Create Langchain Output Parser Structured Node: "JSON Parser: Commentary"**  
    - Type: Langchain Output Parser Structured  
    - Configure to parse structured commentary output.

24. **Connect "AI: Summarize Top 15 Articles" → "Add Expert Commentary"**  
    **Connect "Groq Commentary Model" → "Add Expert Commentary" (language model input)**  
    **Connect "JSON Parser: Commentary" → "Add Expert Commentary" (output parser input)**

25. **Create SplitOut Node: "Split Analysis Results"**  
    - Type: SplitOut  
    - Splits combined commentary results for further processing.

26. **Connect "Add Expert Commentary" → "Split Analysis Results"**

27. **Create Code Node: "Transform for Sheets"**  
    - Type: Code  
    - Write code to transform data into Google Sheets row format.

28. **Connect "Split Analysis Results" → "Transform for Sheets"**

29. **Create Google Sheets Node: "Save to Google Sheets"**  
    - Type: Google Sheets  
    - Configure Google OAuth2 credentials with Sheets API access.  
    - Select target spreadsheet and worksheet.

30. **Connect "Transform for Sheets" → "Save to Google Sheets"**

31. **Create Google Sheets Node: "Archive All Links"**  
    - Type: Google Sheets  
    - Configure to archive URLs of all classified articles.  
    - Use same or separate spreadsheet/worksheet.

32. **Connect "AI: Classify Finance/Insurance" → "Archive All Links"**

33. **Create Code Node: "Format Email (Desktop)"**  
    - Type: Code  
    - Write code to format report email optimized for desktop.

34. **Connect "Save to Google Sheets" → "Format Email (Desktop)"**

35. **Create Gmail Node: "Send Report Email"**  
    - Type: Gmail  
    - Configure Gmail OAuth2 credentials.  
    - Setup recipient(s), subject, and email body from "Format Email (Desktop)".

36. **Connect "Format Email (Desktop)" → "Send Report Email"**

37. *(Optional)* Create and configure "Format Email (Mobile)" as a disabled node for mobile formatting.

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                      |
|--------------------------------------------------------------------------------------------------------------|------------------------------------|
| The workflow uses both OpenAI and Groq AI language models for classification, summarization, and commentary. | Requires API keys and credential setup for both providers. |
| Error handling is designed to continue processing in case of partial failures (e.g., RSS fetch failure).      | Improves robustness for flaky sources. |
| Google Sheets nodes require OAuth2 credentials with sufficient permissions to read/write sheets data.        | Refer to Google Sheets API documentation. |
| Gmail node requires OAuth2 credentials and correct setup for sending emails via Gmail SMTP.                   | See Gmail API and OAuth2 setup guides. |
| The summarization limits to top 15 articles to control token usage and response size.                         | Helps manage cost and performance of OpenAI usage. |
| Disabled mobile email formatting node is present for potential future use or mobile-specific customization.   | Can be enabled and configured as needed. |

---

**Disclaimer:** The provided text is derived solely from an automated n8n workflow. It adheres strictly to current content policies and contains no illegal, offensive, or protected elements. All data processed are legal and public.