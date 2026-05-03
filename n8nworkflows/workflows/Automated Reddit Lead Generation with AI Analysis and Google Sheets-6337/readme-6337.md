Automated Reddit Lead Generation with AI Analysis and Google Sheets

https://n8nworkflows.xyz/workflows/automated-reddit-lead-generation-with-ai-analysis-and-google-sheets-6337


# Automated Reddit Lead Generation with AI Analysis and Google Sheets

---

### 1. Workflow Overview

This workflow automates lead generation on Reddit by leveraging AI analysis and Google Sheets integration. It is designed for businesses seeking to identify high-potential leads from Reddit discussions relevant to their services. The workflow orchestrates data retrieval, AI-powered subreddit selection, query generation, Reddit search, lead classification, opportunity analysis, and lead storage.

Logical blocks:

- **1.1 Trigger and Input Retrieval**: Periodic execution and business profile data loading from Google Sheets.  
- **1.2 Strategic Subreddit Selection**: AI analysis to identify the most relevant subreddits for lead generation.  
- **1.3 Query Generation**: AI generates multiple focused search queries tailored to different lead intent levels.  
- **1.4 Reddit Search Execution**: Searches Reddit with generated queries to retrieve posts.  
- **1.5 Lead Classification and Analysis**: AI classifies and analyzes each Reddit post for lead quality and opportunity.  
- **1.6 High-Value Lead Filtering and Storage**: Filter leads by score and save qualified leads to Google Sheets.

---

### 2. Block-by-Block Analysis

#### 2.1 Trigger and Input Retrieval

**Overview:**  
This block triggers the workflow every 2 hours and retrieves the enhanced business profile from a Google Sheet to provide structured input data for downstream AI processing.

**Nodes Involved:**  
- Schedule Trigger  
- Get Enhanced Business Profile

**Node Details:**  

- **Schedule Trigger**  
  - Type: Schedule Trigger  
  - Role: Initiates workflow every 2 hours.  
  - Configuration: Interval set at 2 hours.  
  - Inputs: None  
  - Outputs: Connects to "Get Enhanced Business Profile".  
  - Edge Cases: Possible downtime if n8n instance is offline; timing skew may cause overlapping executions if workflow runs long.

- **Get Enhanced Business Profile**  
  - Type: Google Sheets  
  - Role: Reads business profile data sheet containing profession, industry, services, client profiles, pain points, etc.  
  - Configuration: Reads from Google Sheet ID "1pmoW6LBeR1E23qfObMf_ce8pOpohP0qmftU9N7Vuq6o", Sheet1 (gid=0). Uses OAuth2 credentials.  
  - Inputs: Trigger from Schedule Trigger.  
  - Outputs: Business profile JSON to "Strategic Subreddit Selector".  
  - Edge Cases: Google Sheets API quota limits, OAuth token expiration, missing or malformed data.

---

#### 2.2 Strategic Subreddit Selection

**Overview:**  
Uses AI (Google Gemini chat model) to analyze the business profile and select the 5 most relevant subreddits for lead generation. Outputs structured JSON with subreddit list and reasoning.

**Nodes Involved:**  
- Google Gemini Chat Model  
- Subreddit Output Parser  
- Strategic Subreddit Selector

**Node Details:**  

- **Google Gemini Chat Model**  
  - Type: AI Language Model (Google Gemini)  
  - Role: Runs prompt to analyze business profile and suggest subreddits.  
  - Configuration: Model "models/gemini-2.0-flash-001", no special options.  
  - Inputs: Business profile JSON.  
  - Outputs: Raw AI response to Subreddit Output Parser.  
  - Edge Cases: API quota, response latency, malformed output.

- **Subreddit Output Parser**  
  - Type: Structured Output Parser  
  - Role: Parses AI response into structured JSON with subreddits array and reasoning string.  
  - Configuration: JSON schema example provided specifying expected output format.  
  - Inputs: AI raw response.  
  - Outputs: Parsed structured data to "Strategic Subreddit Selector".  
  - Edge Cases: Parsing errors if AI response deviates from schema.

- **Strategic Subreddit Selector**  
  - Type: AI Chain LLM  
  - Role: Wraps the prompt and output parser to produce final structured output.  
  - Configuration: Prompt includes business profile fields with selection criteria and avoidance rules.  
  - Inputs: Parsed business profile data.  
  - Outputs: Structured subreddit selection to "Multi-Query Generator".  
  - Edge Cases: Expression evaluation errors in prompt, API errors.

---

#### 2.3 Query Generation

**Overview:**  
Generates 3 targeted Reddit search queries based on business profile and selected subreddits, each addressing different intent levels: direct service requests, problem-focused, and recommendation requests.

**Nodes Involved:**  
- Query Generator Model  
- Multi-Query Output Parser  
- Multi-Query Generator  
- Code (Python)

**Node Details:**  

- **Query Generator Model**  
  - Type: AI Language Model (Google Gemini)  
  - Role: Generates raw query text based on prompt.  
  - Configuration: Model "models/gemini-2.0-flash".  
  - Inputs: Business profile + selected subreddits.  
  - Outputs: Raw query text to Multi-Query Output Parser.  
  - Edge Cases: API issues, malformed generation.

- **Multi-Query Output Parser**  
  - Type: Structured Output Parser  
  - Role: Parses AI response into an array of query objects with type, query string, and intent level.  
  - Configuration: JSON schema example with 3 query entries.  
  - Inputs: Raw AI output.  
  - Outputs: Structured queries to Multi-Query Generator.  
  - Edge Cases: Parsing errors.

- **Multi-Query Generator**  
  - Type: AI Chain LLM  
  - Role: Combines prompt and output parser to finalize query generation.  
  - Configuration: Dynamic prompt uses business profile, selected subreddits, and query rules.  
  - Inputs: Subreddit list from previous block.  
  - Outputs: Structured queries to "Code" node.  
  - Edge Cases: Expression syntax errors, API errors.

- **Code (Python)**  
  - Type: Code node (Python)  
  - Role: Processes the structured query output to adapt format for batch processing. It extracts the "output" field from JSON.  
  - Configuration: Python script loops over input items and returns query list.  
  - Inputs: Structured queries from Multi-Query Generator.  
  - Outputs: Passes queries to "Split Query Batches".  
  - Edge Cases: Script errors if input structure changes.

---

#### 2.4 Reddit Search Execution

**Overview:**  
Splits queries into batches, executes Reddit searches for each query, and processes results for classification.

**Nodes Involved:**  
- Split Query Batches (first instance)  
- Reddit Search Engine  
- Split Query Batches (second instance)

**Node Details:**  

- **Split Query Batches (first instance)**  
  - Type: Split In Batches  
  - Role: Iterates over queries, processing them one by one or in small batches.  
  - Configuration: Default batch size (not specified), reset disabled.  
  - Inputs: Queries from Code node.  
  - Outputs: Individual query to Reddit Search Engine and AI Lead Classifier.  
  - Edge Cases: Batch state reset issues, large batch sizes causing timeouts.

- **Reddit Search Engine**  
  - Type: Reddit node  
  - Role: Performs Reddit search using the query string.  
  - Configuration: Limit 10 results per query, sorted by "new", location "allReddit". OAuth2 credentials used for Reddit API access.  
  - Inputs: Query string from batch splitter.  
  - Outputs: Search results to "Split Query Batches" (second instance).  
  - Edge Cases: Reddit API rate limits, authentication errors, empty search results.

- **Split Query Batches (second instance)**  
  - Type: Split In Batches  
  - Role: Splits Reddit search results into individual posts for processing.  
  - Configuration: Default batch size, reset disabled.  
  - Inputs: Reddit search results.  
  - Outputs: Individual Reddit posts to AI Lead Classifier.  
  - Edge Cases: Batch state issues, empty results.

---

#### 2.5 Lead Classification and Analysis

**Overview:**  
Classifies Reddit posts by lead potential and performs detailed service opportunity analysis using AI models for each post.

**Nodes Involved:**  
- AI Lead Classifier  
- Loop Over Items  
- Service Opportunity Analyzer  
- Service Analysis Parser  
- High Value Filter

**Node Details:**  

- **AI Lead Classifier**  
  - Type: AI Text Classifier (Google Gemini)  
  - Role: Classifies each Reddit post into categories: HIGH_POTENTIAL, MEDIUM_POTENTIAL, LOW_POTENTIAL, NO_POTENTIAL.  
  - Configuration: Input text compiled from business profile and Reddit post metadata/content. Categories defined with descriptions and criteria.  
  - Inputs: Individual Reddit posts.  
  - Outputs: Classified posts to "Loop Over Items".  
  - Edge Cases: Misclassification, API latency.

- **Loop Over Items**  
  - Type: Split In Batches  
  - Role: Iterates over classified posts for further analysis.  
  - Configuration: Default batch size, reset disabled.  
  - Inputs: Classified posts.  
  - Outputs: Individual posts to "Service Opportunity Analyzer".  
  - Edge Cases: Batch handling issues.

- **Service Opportunity Analyzer**  
  - Type: AI Chain LLM (Google Gemini)  
  - Role: Analyzes each Reddit post for specific service opportunities, lead scoring, urgency, and outreach recommendations.  
  - Configuration: Prompt includes professional context, Reddit post details, and a scoring framework with multiple criteria and output requirements. Uses structured output parser.  
  - Inputs: Individual posts from Loop Over Items.  
  - Outputs: Structured analysis JSON to "High Value Filter".  
  - Edge Cases: API errors, output parsing failures.

- **Service Analysis Parser**  
  - Type: Structured Output Parser  
  - Role: Parses the AI service opportunity analysis into structured JSON with lead score, urgency, problem, recommended service, and outreach approach.  
  - Configuration: JSON schema example provided.  
  - Inputs: AI raw output.  
  - Outputs: Parsed data to "High Value Filter".  
  - Edge Cases: Parsing errors.

- **High Value Filter**  
  - Type: Filter node  
  - Role: Filters analyzed leads to only those with a lead score >= 6 (high-value leads).  
  - Configuration: Numeric condition on property output.lead_score >= 6.  
  - Inputs: Parsed analysis.  
  - Outputs: Filtered leads to "Save High-Value Leads" node.  
  - Edge Cases: Incorrect data types, missing lead_score.

---

#### 2.6 High-Value Lead Storage

**Overview:**  
Stores qualified high-value leads into a dedicated Google Sheet for tracking and follow-up.

**Nodes Involved:**  
- Save High-Value Leads  
- Loop Over Items (second instance)

**Node Details:**  

- **Save High-Value Leads**  
  - Type: Google Sheets  
  - Role: Appends or updates rows in a Google Sheet with lead data including post ID, URL, date, content, subreddit, and service recommendations.  
  - Configuration: Google Sheet ID "1cTkC1xkj11B0O81UBz-DXbkZ9oAOxsBTk3YsHJHfJ1k", Sheet1 (gid=0). Uses OAuth2 credentials. Uses "post_id" as matching column for updates.  
  - Inputs: Filtered high-value leads.  
  - Outputs: Feeds back into "Loop Over Items" (second instance) to process next lead.  
  - Edge Cases: API limits, credential expiration, data conflicts on update.

- **Loop Over Items (second instance)**  
  - Type: Split In Batches  
  - Role: Iterates over high-value leads to feed into Save High-Value Leads node.  
  - Configuration: Default batching, reset disabled.  
  - Inputs: High-value leads from filter.  
  - Outputs: Single lead per iteration to Save High-Value Leads.  
  - Edge Cases: Batch mismanagement.

---

### 3. Summary Table

| Node Name                   | Node Type                               | Functional Role                        | Input Node(s)                  | Output Node(s)               | Sticky Note                                                  |
|-----------------------------|---------------------------------------|-------------------------------------|-------------------------------|------------------------------|--------------------------------------------------------------|
| Schedule Trigger            | Schedule Trigger                      | Initiate workflow every 2 hours     | -                             | Get Enhanced Business Profile |                                                              |
| Get Enhanced Business Profile| Google Sheets                        | Load business profile data           | Schedule Trigger              | Strategic Subreddit Selector  |                                                              |
| Google Gemini Chat Model    | AI Language Model (Google Gemini)    | Generate subreddit suggestions       | Get Enhanced Business Profile | Strategic Subreddit Selector  |                                                              |
| Subreddit Output Parser     | Output Parser (structured)            | Parse subreddit list from AI output  | Google Gemini Chat Model      | Strategic Subreddit Selector  |                                                              |
| Strategic Subreddit Selector| AI Chain LLM                         | Select top 5 relevant subreddits     | Get Enhanced Business Profile | Multi-Query Generator         |                                                              |
| Query Generator Model       | AI Language Model (Google Gemini)    | Generate search queries               | Strategic Subreddit Selector  | Multi-Query Generator         |                                                              |
| Multi-Query Output Parser   | Output Parser (structured)            | Parse generated queries               | Query Generator Model         | Multi-Query Generator         |                                                              |
| Multi-Query Generator       | AI Chain LLM                         | Generate 3 focused queries            | Strategic Subreddit Selector  | Code                         |                                                              |
| Code                       | Code (Python)                       | Extract and format queries            | Multi-Query Generator         | Split Query Batches (1st)     |                                                              |
| Split Query Batches (1st)    | Split In Batches                    | Iterate over generated queries        | Code                         | Reddit Search Engine, AI Lead Classifier |                                                              |
| Reddit Search Engine        | Reddit                              | Execute Reddit search for queries    | Split Query Batches (1st)     | Split Query Batches (2nd)     |                                                              |
| Split Query Batches (2nd)    | Split In Batches                    | Iterate over Reddit search results    | Reddit Search Engine          | AI Lead Classifier            |                                                              |
| AI Lead Classifier          | AI Text Classifier (Google Gemini)  | Classify posts by lead potential      | Split Query Batches (2nd)     | Loop Over Items (1st)         |                                                              |
| Loop Over Items (1st)        | Split In Batches                    | Iterate over classified posts         | AI Lead Classifier            | Service Opportunity Analyzer  |                                                              |
| Service Opportunity Analyzer| AI Chain LLM                       | Analyze posts for lead scoring & service opportunities | Loop Over Items (1st)         | High Value Filter             |                                                              |
| Service Analysis Parser     | Output Parser (structured)            | Parse AI analysis into structured data| Service Opportunity Analyzer  | High Value Filter             |                                                              |
| High Value Filter           | Filter                              | Filter leads with lead_score >= 6    | Service Opportunity Analyzer  | Save High-Value Leads         |                                                              |
| Save High-Value Leads       | Google Sheets                      | Save qualified leads to spreadsheet  | High Value Filter             | Loop Over Items (2nd)         |                                                              |
| Loop Over Items (2nd)        | Split In Batches                    | Iterate over high-value leads         | Save High-Value Leads         | Save High-Value Leads         |                                                              |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Schedule Trigger**  
   - Type: Schedule Trigger  
   - Set interval to every 2 hours.

2. **Add Google Sheets node: Get Enhanced Business Profile**  
   - Connect from Schedule Trigger.  
   - Set Document ID: "1pmoW6LBeR1E23qfObMf_ce8pOpohP0qmftU9N7Vuq6o"  
   - Sheet Name: "Sheet1" (gid=0)  
   - Configure Google Sheets OAuth2 credentials.

3. **Add Google Gemini Chat Model node: Subreddit Suggestion**  
   - Connect from "Get Enhanced Business Profile".  
   - Model: "models/gemini-2.0-flash-001"  
   - Prompt: Provide business profile fields and ask for 5 relevant subreddits with selection criteria and avoidance rules.

4. **Add Structured Output Parser: Subreddit Output Parser**  
   - Connect from Google Gemini Chat Model.  
   - Provide JSON schema example with "subreddits" array and "reasoning" string.

5. **Add AI Chain LLM: Strategic Subreddit Selector**  
   - Use prompt with embedded business profile fields and criteria.  
   - Enable output parser using above node.  
   - Connect from "Get Enhanced Business Profile" and "Subreddit Output Parser" outputs.

6. **Add Google Gemini Chat Model: Query Generator Model**  
   - Connect from "Strategic Subreddit Selector".  
   - Model: "models/gemini-2.0-flash"  
   - Prompt: Use business profile and subreddit list to generate 3 queries of different intent levels.

7. **Add Structured Output Parser: Multi-Query Output Parser**  
   - Connect from Query Generator Model.  
   - JSON schema example with array of query objects (type, query string, intent level).

8. **Add AI Chain LLM: Multi-Query Generator**  
   - Connect from "Strategic Subreddit Selector".  
   - Use prompt for generating 3 focused queries with rules on keywords, length, and intent.  
   - Enable output parser with Multi-Query Output Parser.

9. **Add Code node (Python)**  
   - Connect from Multi-Query Generator.  
   - Python code to extract queries from JSON output for batching.

10. **Add Split In Batches node: Split Query Batches (1st)**  
    - Connect from Code node.  
    - Use default batch size.

11. **Add Reddit node: Reddit Search Engine**  
    - Connect from Split Query Batches (1st).  
    - Operation: Search  
    - Keyword: Use current batch query string  
    - Limit: 10  
    - Sort: new  
    - Credentials: Reddit OAuth2.

12. **Add Split In Batches node: Split Query Batches (2nd)**  
    - Connect from Reddit Search Engine.  
    - Use default batch size.

13. **Add AI Text Classifier node: AI Lead Classifier**  
    - Connect from Split Query Batches (2nd).  
    - Configure text input with business profile and Reddit post metadata/content.  
    - Categories: HIGH_POTENTIAL, MEDIUM_POTENTIAL, LOW_POTENTIAL, NO_POTENTIAL with descriptions.

14. **Add Split In Batches node: Loop Over Items (1st)**  
    - Connect from AI Lead Classifier.

15. **Add AI Chain LLM node: Service Opportunity Analyzer**  
    - Connect from Loop Over Items (1st).  
    - Prompt includes professional context and Reddit post details with scoring framework.  
    - Enable output parser.

16. **Add Structured Output Parser node: Service Analysis Parser**  
    - Connect from Service Opportunity Analyzer.

17. **Add Filter node: High Value Filter**  
    - Connect from Service Opportunity Analyzer.  
    - Condition: output.lead_score >= 6.

18. **Add Google Sheets node: Save High-Value Leads**  
    - Connect from High Value Filter.  
    - Document ID: "1cTkC1xkj11B0O81UBz-DXbkZ9oAOxsBTk3YsHJHfJ1k"  
    - Sheet1 (gid=0)  
    - Mapping columns: Include post_id, url, date, title, subreddit, post content, and service suggestion.  
    - Use append or update mode with post_id as key.  
    - Configure Google Sheets OAuth2 credentials.

19. **Add Split In Batches node: Loop Over Items (2nd)**  
    - Connect from Save High-Value Leads.  
    - Feeds back into Save High-Value Leads node for batch processing.

---

### 5. General Notes & Resources

| Note Content                                                                                   | Context or Link                                                                                   |
|------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow demonstrates advanced integration of Google Gemini (PaLM) AI models within n8n.      | AI-powered subreddit and query generation, lead classification, and opportunity analysis.       |
| Uses Google Sheets for both input profile data and lead storage to maintain centralized data. | Google Sheets API OAuth2 credentials required for read/write operations.                         |
| Reddit API OAuth2 credentials required for search operations to avoid rate limits and bans.    | Reddit OAuth2 app setup needed with proper scopes for search.                                   |
| AI prompt engineering is critical for relevant subreddit selection and lead query formulation. | Prompts include clear instructions on business context and lead intent signals.                 |
| Batch processing nodes ensure scalability but require attention to batch sizes and reset flags.| Misconfiguration may cause data skipping or infinite loops.                                    |
| Output parsers with strict JSON schemas reduce errors in AI response handling.                 | Ensure AI responses conform to expected schemas; failures may halt workflow.                     |
| The workflow is designed to run periodically, but error handling and rate limit retries are recommended for production. | Consider adding retry nodes or error workflows to handle API failures gracefully.               |

---

Disclaimer: The provided text is exclusively derived from an automated workflow created with n8n, an integration and automation tool. This processing strictly respects current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and publicly accessible.

---