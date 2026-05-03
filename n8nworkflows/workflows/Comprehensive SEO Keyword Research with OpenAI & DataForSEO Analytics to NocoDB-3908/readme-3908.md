Comprehensive SEO Keyword Research with OpenAI & DataForSEO Analytics to NocoDB

https://n8nworkflows.xyz/workflows/comprehensive-seo-keyword-research-with-openai---dataforseo-analytics-to-nocodb-3908


# Comprehensive SEO Keyword Research with OpenAI & DataForSEO Analytics to NocoDB

### 1. Workflow Overview

This n8n workflow automates comprehensive SEO keyword research by integrating AI-driven keyword generation with data analytics from DataForSEO, and stores results in NocoDB. It is designed to streamline the generation of actionable keyword strategies and content briefs, triggered by input data from NocoDB, and enhanced with competitor insights and keyword metrics.

The workflow consists of the following logical blocks:

- **1.1 Input Reception & Initialization**: Receives SEO research requests from NocoDB via webhook, extracts and formats input parameters, and updates request status.
- **1.2 Topic Expansion via AI**: Uses OpenAI to generate structured keyword ideas and related topics based on input parameters.
- **1.3 Keyword Metrics Analysis**: Queries DataForSEO APIs to retrieve search volume, CPC, and keyword difficulty for the generated keywords.
- **1.4 Competitor Analysis**: Analyzes competitor URLs to identify their ranking keywords and detect content gaps.
- **1.5 Final Strategy Generation**: Combines all gathered data and uses OpenAI to produce a detailed SEO keyword strategy and content brief.
- **1.6 Output Storage & Notifications**: Saves the final strategy to NocoDB, updates input status, and sends Slack notifications on workflow start and completion.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception & Initialization

**Overview**:  
This block triggers the workflow upon receiving a POST request from NocoDB, extracts all necessary input parameters, formats competitor URLs, and updates the workflow status to “Started”. It also sends a Slack notification to indicate the start of processing.

**Nodes Involved**:  
- Get Input from NocoDB (Webhook)  
- Set relevant fields  
- Format Json and add Competitor URLs (Code)  
- Split the Competitor URLs (SplitOut)  
- Update Status - Started (NocoDB Update)  
- Start Notification (Slack)  
- Sticky Note (Input)  
- Sticky Note (Notification and Update Status)

**Node Details**:

- **Get Input from NocoDB**  
  - *Type*: Webhook (HTTP POST)  
  - *Role*: Entry point for new keyword research requests from NocoDB  
  - *Configuration*: Listens on a unique webhook path, expecting POST with structured data  
  - *Input/Output*: Outputs JSON payload with input data fields  
  - *Potential Failures*: Network errors, invalid payload format, webhook unavailability

- **Set relevant fields**  
  - *Type*: Set node  
  - *Role*: Extracts relevant fields from webhook payload for use downstream (primary topic, competitor URLs, target audience, etc.)  
  - *Configuration*: Assigns multiple string fields from nested JSON path `$json.body.data.rows[0]`  
  - *Key Expressions*: Uses expressions like `={{ $json.body.data.rows[0]['Primary Topic'] }}`  
  - *Input/Output*: Inputs webhook JSON; outputs flattened, named fields for easier access  
  - *Edge Cases*: Missing or malformed fields could result in empty values downstream

- **Format Json and add Competitor URLs**  
  - *Type*: Code (JavaScript)  
  - *Role*: Splits the competitor URLs string into a trimmed array  
  - *Code Logic*: Splits string by commas, trims whitespace, filters empty entries  
  - *Input/Output*: Expects `competitor_urls` string, outputs array `competitorUrls`  
  - *Failures*: Empty or malformed URL string; no URLs parsed

- **Split the Competitor URLs**  
  - *Type*: SplitOut  
  - *Role*: Splits the array of competitor URLs into individual items for iterative processing  
  - *Input/Output*: Takes array `competitorUrls`, outputs one URL per item  
  - *Edge Cases*: Empty array results in no downstream competitor analysis calls

- **Update Status - Started**  
  - *Type*: NocoDB (Update)  
  - *Role*: Marks the input record’s status as "Started" in NocoDB  
  - *Configuration*: Updates row by `Id`, sets `Status` field  
  - *Credentials*: Requires NocoDB API Token with update permissions  
  - *Failures*: API auth error, network issues, invalid Id

- **Start Notification**  
  - *Type*: Slack  
  - *Role*: Sends notification to designated Slack channel indicating research start  
  - *Configuration*: Uses Slack API token credential, posts a message with key parameters  
  - *Channel*: SEO Keyword Research channel (ID cached)  
  - *Failures*: Slack API rate limit, invalid token

- **Sticky Note (Input and Notification)**  
  - *Type*: Sticky Note  
  - *Role*: Provides visual grouping and documentation for this block in the editor

---

#### 2.2 Topic Expansion via AI

**Overview**:  
Generates structured keyword data including primary keywords, long-tail variations, question-based keywords, and related topics using OpenAI’s language model, with JSON structured output parsing.

**Nodes Involved**:  
- Topic Expansion (Langchain Agent)  
- OpenAI Chat Model (Langchain LLM)  
- Structured Output Parser (Langchain Output Parser)  
- split primary keywords (SplitOut)  
- Sticky Note (Topic Expansion)

**Node Details**:

- **Topic Expansion**  
  - *Type*: Langchain Agent Node (OpenAI LLM interface)  
  - *Role*: Sends a prompt to OpenAI requesting generation of keyword lists and related topics based on input fields  
  - *Prompt*: Template referencing input variables such as `primary_topic`, `target_audience`, `content_type`, etc., requesting a structured JSON output with categories: primary_keywords, long_tail_keywords (with intent), question_based_keywords, related_topics  
  - *Output Parser*: Connected to Structured Output Parser node for JSON validation and extraction  
  - *Failures*: Model timeout, malformed prompt, output parsing errors

- **OpenAI Chat Model**  
  - *Type*: Langchain LLM (OpenAI Chat)  
  - *Role*: Executes the actual chat completion request with specified model (e.g., GPT-4 or GPT-3.5)  
  - *Credentials*: OpenAI API key required  
  - *Failures*: API quota exceeded, authentication failures

- **Structured Output Parser**  
  - *Type*: Langchain Output Parser  
  - *Role*: Validates and parses OpenAI response into structured JSON following the defined schema  
  - *Schema*: Includes arrays of strings and objects with "keyword" and "intent" fields  
  - *Failures*: Invalid or incomplete JSON, parsing errors

- **split primary keywords**  
  - *Type*: SplitOut  
  - *Role*: Splits the array of primary keywords for individual metric queries downstream  
  - *Failures*: Empty keyword array results in no metric queries

- **Sticky Note (Topic Expansion)**  
  - *Type*: Sticky Note  
  - *Role*: Documentation and grouping for the topic expansion logic

---

#### 2.3 Keyword Metrics Analysis

**Overview**:  
Queries DataForSEO’s API to fetch keyword search volume, cost per click (CPC), and keyword difficulty metrics for primary keywords.

**Nodes Involved**:  
- Search Volume & CPC (DataForSEO node)  
- Keyword Difficulty (DataForSEO node)  
- Aggregate SV & CPC (Aggregate)  
- Aggregate KWD (Aggregate)  
- Merge SV, CPC & KWD (Merge)  
- Merge Topic Expansion, SV, CPC & KWD (Merge)  
- Sticky Note (Search Volume, Cost Per Click, Keyword Difficulty)

**Node Details**:

- **Search Volume & CPC**  
  - *Type*: DataForSEO API node  
  - *Role*: Retrieves monthly search volume and CPC data per keyword  
  - *Input*: Receives primary keywords array, language, and location parameters  
  - *Configuration*: Uses DataForSEO credentials, API resource `keywords_data`  
  - *Failures*: API auth errors, rate limiting, invalid parameters, empty keyword list

- **Keyword Difficulty**  
  - *Type*: DataForSEO API node  
  - *Role*: Gets keyword difficulty scores indicating ranking competitiveness  
  - *Input*: Same keywords, language, and location as above  
  - *Resource*: `labs/get-keyword-difficulty`  
  - *Failures*: Same as above

- **Aggregate SV & CPC** and **Aggregate KWD**  
  - *Type*: Aggregate nodes  
  - *Role*: Collate multiple API response items into a single aggregated dataset for easier merging  
  - *Failures*: No input items cause empty aggregation

- **Merge SV, CPC & KWD**  
  - *Type*: Merge node  
  - *Role*: Combines aggregated SV & CPC data with KWD data by matching positions  
  - *Mode*: Combine by position (array index)  
  - *Failures*: Mismatch in array lengths leads to incomplete merges

- **Merge Topic Expansion, SV, CPC & KWD**  
  - *Type*: Merge node  
  - *Role*: Combines the original Topic Expansion JSON output with the keyword metrics data  
  - *Failures*: Position mismatch, missing data

- **Sticky Note (Search Volume, CPC, Keyword Difficulty)**  
  - *Role*: Documentation for this metrics analysis phase

---

#### 2.4 Competitor Analysis

**Overview**:  
Analyzes competitor URLs from input to identify their ranking keywords and content gaps by querying DataForSEO and summarizing with an AI agent.

**Nodes Involved**:  
- Keyword Ranking per URL (DataForSEO node)  
- OpenAI Chat Model1 (Langchain LLM)  
- Competitor Analysis (Langchain Agent)  
- Aggregate Competitor Analysis (Aggregate)  
- Merge Everything (Merge)  
- Sticky Note (Competitor Research)

**Node Details**:

- **Keyword Ranking per URL**  
  - *Type*: DataForSEO API node  
  - *Role*: Retrieves the top ranked keywords for each competitor URL  
  - *Input*: Competitor URLs, language, and location parameters  
  - *Operation*: `labs/get-ranked-keywords` with limit 10 keywords per URL  
  - *Failures*: Invalid URLs, API errors

- **Aggregate Competitor Analysis**  
  - *Type*: Aggregate node  
  - *Role*: Aggregates keyword rankings from all competitor URLs into a single dataset

- **OpenAI Chat Model1**  
  - *Type*: Langchain LLM node  
  - *Role*: Executes the OpenAI LLM request for competitor analysis synthesis

- **Competitor Analysis**  
  - *Type*: Langchain Agent node  
  - *Role*: Receives competitor keyword data and analyzes it to identify primary keywords, content gaps, unique angles, and questions addressed by competitors  
  - *Prompt*: Structured text referencing competitor URLs, primary topic, and competitor keyword data  
  - *Output*: Structured JSON analysis  
  - *Failures*: Model errors, parsing errors

- **Merge Everything**  
  - *Type*: Merge node  
  - *Role*: Combines competitor analysis results with prior merged keyword and metric data for final synthesis  
  - *Inputs*: From topic expansion + metrics merge and competitor analysis aggregate  
  - *Failures*: Position or key mismatch

- **Sticky Note (Competitor Research)**  
  - *Role*: Visual documentation for competitor analysis phase

---

#### 2.5 Final Strategy Generation

**Overview**:  
Uses OpenAI to synthesize all keyword data, metrics, and competitor insights into a detailed SEO keyword strategy and actionable content brief in Markdown format.

**Nodes Involved**:  
- Final Keyword Strategy (Langchain Agent)  
- OpenAI Chat Model2 (Langchain LLM)  
- Write Content Brief (NocoDB Create)  
- Sticky Note (Merge and write Final Keyword Strategy)

**Node Details**:

- **Final Keyword Strategy**  
  - *Type*: Langchain Agent node  
  - *Role*: Receives combined data (primary keywords, metrics, competitor insights) and generates a comprehensive SEO strategy and content plan  
  - *Prompt*: Detailed instruction including sections for executive summary, keyword strategy, competitive landscape, content outline, and SEO titles; outputs Markdown-ready text  
  - *Failures*: Model timeouts, incomplete output, prompt errors

- **OpenAI Chat Model2**  
  - *Type*: Langchain LLM node  
  - *Role*: Runs the final OpenAI completion call with the above prompt  
  - *Credentials*: OpenAI API key  
  - *Failures*: API errors, quota issues

- **Write Content Brief**  
  - *Type*: NocoDB (Create)  
  - *Role*: Saves the generated SEO keyword strategy markdown content back into the output table in NocoDB  
  - *Fields*: `primary_topic_used`, `report_content` (Markdown), automatic timestamp by NocoDB  
  - *Failures*: API auth, network failures

- **Sticky Note (Merge and write Final Keyword Strategy)**  
  - *Role*: Documentation for this final generation and storage block

---

#### 2.6 Output Storage & Notifications

**Overview**:  
Updates the input record’s status to "Done" in NocoDB and sends a Slack notification confirming workflow completion.

**Nodes Involved**:  
- Update Status - Done (NocoDB Update)  
- Send Notification (Slack)  
- Sticky Note (Save, Update Status and Notify)

**Node Details**:

- **Update Status - Done**  
  - *Type*: NocoDB (Update)  
  - *Role*: Marks the input record’s status as “Done” to indicate completion  
  - *Configuration*: Uses input record Id, sets `Status` = "Done"  
  - *Failures*: API auth or network errors

- **Send Notification**  
  - *Type*: Slack  
  - *Role*: Sends a detailed message to Slack channel summarizing the completed SEO keyword research  
  - *Message*: Includes primary topic, audience, content type, location, language, competitor URLs  
  - *Failures*: Slack API or network issues

- **Sticky Note (Save, Update Status and Notify)**  
  - *Role*: Visual grouping and documentation

---

### 3. Summary Table

| Node Name                        | Node Type                            | Functional Role                                  | Input Node(s)                             | Output Node(s)                                 | Sticky Note                                              |
|---------------------------------|------------------------------------|-------------------------------------------------|------------------------------------------|------------------------------------------------|----------------------------------------------------------|
| Get Input from NocoDB            | Webhook                            | Receives input from NocoDB webhook               | —                                        | Set relevant fields                             | Input                                                    |
| Set relevant fields              | Set                                | Extracts and formats input parameters            | Get Input from NocoDB                     | Topic Expansion, Format Json and add Competitor URLs, Merge Everything, Update Status - Started, Start Notification | Input                                                    |
| Format Json and add Competitor URLs | Code                              | Parses competitor URLs string into array         | Set relevant fields                       | Split the Competitor URLs                       | Input                                                    |
| Split the Competitor URLs        | SplitOut                          | Splits competitor URLs array into individual URLs | Format Json and add Competitor URLs      | Keyword Ranking per URL                         | Input                                                    |
| Update Status - Started          | NocoDB Update                     | Updates input status to "Started"                 | Set relevant fields                       | —                                              | Notification and Update Status                           |
| Start Notification              | Slack                             | Sends Slack notification on workflow start       | Set relevant fields                       | —                                              | Notification and Update Status                           |
| Topic Expansion                 | Langchain Agent                   | Generates structured keywords with AI             | Set relevant fields                       | split primary keywords                          | Topic Expansion                                         |
| OpenAI Chat Model               | Langchain LLM                    | Executes keyword generation prompt                | —                                        | Topic Expansion                                 | Topic Expansion                                         |
| Structured Output Parser        | Langchain Output Parser          | Parses AI output into structured JSON             | Topic Expansion                          | Topic Expansion                                 | Topic Expansion                                         |
| split primary keywords          | SplitOut                         | Splits primary keywords for metric analysis       | Topic Expansion                          | Search Volume & CPC, Keyword Difficulty        | Search Volume, Cost Per Click, Keyword Difficulty      |
| Search Volume & CPC             | DataForSEO API                   | Fetches search volume and CPC metrics             | split primary keywords                   | Aggregate SV & CPC                              | Search Volume, Cost Per Click, Keyword Difficulty      |
| Keyword Difficulty             | DataForSEO API                   | Fetches keyword difficulty metrics                 | split primary keywords                   | Aggregate KWD                                   | Search Volume, Cost Per Click, Keyword Difficulty      |
| Aggregate SV & CPC              | Aggregate                        | Aggregates search volume and CPC data              | Search Volume & CPC                      | Merge SV, CPC & KWD                             | Search Volume, Cost Per Click, Keyword Difficulty      |
| Aggregate KWD                  | Aggregate                        | Aggregates keyword difficulty data                 | Keyword Difficulty                      | Merge SV, CPC & KWD                             | Search Volume, Cost Per Click, Keyword Difficulty      |
| Merge SV, CPC & KWD             | Merge                           | Combines SV/CPC data with keyword difficulty       | Aggregate SV & CPC, Aggregate KWD        | Merge Topic Expansion, SV, CPC & KWD            | Search Volume, Cost Per Click, Keyword Difficulty      |
| Merge Topic Expansion, SV, CPC & KWD | Merge                       | Combines topic expansion data with all keyword metrics | Topic Expansion, Merge SV, CPC & KWD     | Merge Everything                                | Search Volume, Cost Per Click, Keyword Difficulty      |
| Keyword Ranking per URL         | DataForSEO API                  | Retrieves ranked keywords for each competitor URL | Split the Competitor URLs                | Competitor Analysis                             | Competitor Research                                   |
| Competitor Analysis            | Langchain Agent                 | Synthesizes competitor keyword data into analysis | OpenAI Chat Model1                      | Aggregate Competitor Analysis                   | Competitor Research                                   |
| OpenAI Chat Model1             | Langchain LLM                  | Executes competitor analysis prompt                | —                                        | Competitor Analysis                             | Competitor Research                                   |
| Aggregate Competitor Analysis   | Aggregate                      | Aggregates competitor keyword ranking data         | Keyword Ranking per URL                  | Merge Everything                                | Competitor Research                                   |
| Merge Everything               | Merge                         | Combines all data sources for final strategy       | Merge Topic Expansion, SV, CPC & KWD, Aggregate Competitor Analysis, Set relevant fields | Final Keyword Strategy                           | Merge and write Final Keyword Strategy                |
| Final Keyword Strategy          | Langchain Agent               | Generates final SEO keyword strategy and brief     | Merge Everything                        | Write Content Brief, Update Status - Done, Send Notification | Merge and write Final Keyword Strategy                |
| OpenAI Chat Model2             | Langchain LLM                | Executes final strategy generation prompt          | —                                        | Final Keyword Strategy                          | Merge and write Final Keyword Strategy                |
| Write Content Brief            | NocoDB Create                | Saves final SEO content brief to NocoDB output     | Final Keyword Strategy                  | —                                              | Save, Update Status and Notify                         |
| Update Status - Done           | NocoDB Update               | Updates input status to "Done"                       | Final Keyword Strategy                  | —                                              | Save, Update Status and Notify                         |
| Send Notification             | Slack                       | Sends Slack notification upon workflow completion  | Final Keyword Strategy                  | —                                              | Save, Update Status and Notify                         |
| Sticky Note                   | Sticky Note                  | Visual documentation node                            | —                                        | —                                              | Various (see above)                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Webhook Node (Get Input from NocoDB)**  
   - Type: Webhook (HTTP POST)  
   - Path: Unique path (e.g., `ac7e989d-6e32-4850-83c4-f10421467fb8`)  
   - This node listens to NocoDB webhook POST requests.

2. **Add Set Node (Set relevant fields)**  
   - Extract and assign fields from the webhook JSON payload:  
     - `primary_topic` = `{{$json.body.data.rows[0]['Primary Topic']}}`  
     - `competitor_urls` = `{{$json.body.data.rows[0]['Competitor URLs']}}`  
     - `target_audience`, `content_type`, `location`, `language`, `status`, `id` similarly  
   - Connect from Webhook node.

3. **Add Code Node (Format Json and add Competitor URLs)**  
   - JavaScript to split competitor URLs string into array:  
     ```js
     const rawUrls = $input.first().json.competitor_urls;
     const competitorUrls = rawUrls.split(",").map(u => u.trim()).filter(u => u.length > 0);
     return [{ json: { ...$input.first().json, competitorUrls } }];
     ```
   - Connect from Set node.

4. **Add SplitOut Node (Split the Competitor URLs)**  
   - Field to split out: `competitorUrls`  
   - Connect from Code node.

5. **Add NocoDB Update Node (Update Status - Started)**  
   - Operation: Update  
   - Table: Input table (e.g., `"mp3qmbuye3pyihc"`)  
   - Fields: `Id` = `{{$json.id}}`, `Status` = `"Started"`  
   - Credentials: NocoDB API Token  
   - Connect from Set node.

6. **Add Slack Node (Start Notification)**  
   - Send a message to the SEO keyword research channel with input details  
   - Use Slack API credentials  
   - Connect from Set node.

7. **Add Langchain Agent (Topic Expansion)**  
   - Prompt: Include input parameters (`primary_topic`, `target_audience`, etc.) requesting structured JSON output of keywords and topics  
   - Has output parser enabled  
   - Connect from Set node.

8. **Add OpenAI Chat Model (OpenAI Chat Model)**  
   - Model: e.g., GPT-4 or GPT-3.5  
   - Credentials: OpenAI API key  
   - Connect as LLM for Topic Expansion node.

9. **Add Structured Output Parser Node**  
   - JSON schema example includes arrays: primary_keywords, long_tail_keywords (with intent), question_based_keywords, related_topics  
   - Connect output parser to Topic Expansion node.

10. **Add SplitOut Node (split primary keywords)**  
    - Field: `output.primary_keywords`  
    - Connect from Topic Expansion node.

11. **Add DataForSEO Node (Search Volume & CPC)**  
    - Resource: `keywords_data`  
    - Operation: default  
    - Keywords: Pass individual primary keywords from split node  
    - Language and Location: Use expressions from `Set relevant fields` node  
    - Credentials: DataForSEO API  
    - Connect from split primary keywords node.

12. **Add DataForSEO Node (Keyword Difficulty)**  
    - Resource: `labs/get-keyword-difficulty`  
    - Keywords, language, location as above  
    - Credentials: DataForSEO API  
    - Connect from split primary keywords node.

13. **Add Aggregate Nodes (Aggregate SV & CPC and Aggregate KWD)**  
    - Aggregate all items from Search Volume & CPC and Keyword Difficulty nodes respectively  
    - Connect from respective DataForSEO nodes.

14. **Add Merge Node (Merge SV, CPC & KWD)**  
    - Mode: Combine  
    - Combine by position (to align metrics per keyword)  
    - Connect from Aggregate SV & CPC and Aggregate KWD nodes.

15. **Add Merge Node (Merge Topic Expansion, SV, CPC & KWD)**  
    - Mode: Combine  
    - Combine by position  
    - Connect from Topic Expansion node and Merge SV, CPC & KWD node.

16. **Add DataForSEO Node (Keyword Ranking per URL)**  
    - Resource: `labs/get-ranked-keywords`  
    - Limit: 10 per URL  
    - Target: Use individual competitor URLs from Split the Competitor URLs node  
    - Language and Location from Set node  
    - Credentials: DataForSEO API  
    - Connect from Split the Competitor URLs node.

17. **Add Aggregate Node (Aggregate Competitor Analysis)**  
    - Aggregate all competitor keyword ranking results  
    - Connect from Keyword Ranking per URL node.

18. **Add Langchain Agent Node (Competitor Analysis)**  
    - Prompt: Analyze competitor keyword rankings, identify primary keywords, content gaps, unique angles, and questions  
    - Connect LLM: OpenAI Chat Model1 (similar config as prior)  
    - Connect input from OpenAI Chat Model1 node.

19. **Add Merge Node (Merge Everything)**  
    - Mode: Combine  
    - Combine by position  
    - Inputs: Merge Topic Expansion, SV, CPC & KWD; Aggregate Competitor Analysis; Set relevant fields (for metadata)  
    - Connect accordingly.

20. **Add Langchain Agent Node (Final Keyword Strategy)**  
    - Prompt: Detailed instructions to synthesize data into an actionable SEO keyword strategy and content brief in Markdown  
    - Connect LLM: OpenAI Chat Model2  
    - Credentials: OpenAI API key  
    - Connect from Merge Everything node.

21. **Add NocoDB Create Node (Write Content Brief)**  
    - Table: Output table (e.g., `"mfsjucjn304v1hc"`)  
    - Fields: `primary_topic_used` = `{{$json.primary_topic}}`, `report_content` = `{{$json.output}}`  
    - Credentials: NocoDB API Token  
    - Connect from Final Keyword Strategy node.

22. **Add NocoDB Update Node (Update Status - Done)**  
    - Table: Input table  
    - Fields: `Id` = `{{$json.id}}`, `Status` = `"Done"`  
    - Credentials: NocoDB API Token  
    - Connect from Final Keyword Strategy node.

23. **Add Slack Node (Send Notification)**  
    - Sends completion notification with summary to Slack channel  
    - Connect from Final Keyword Strategy node.

24. **Add Sticky Notes** to document each logical block and clarify workflow structure visually.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                          | Context or Link                                                                                                     |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Workflow diagram and screenshots provide visual overview of flow and NocoDB table structures.                                                                                                                                                         | https://github.com/philrox/n8n-workflows/blob/main/AI-Powered%20SEO%20Keyword%20Research%20Automation/screenshots/  |
| Original workflow source and inspiration: AI-Powered SEO Keyword Research Automation - The vibe Marketer                                                                                                                                              | https://templates.thevibemarketer.com/template/seo-keyword-research                                                 |
| Requires API credentials for OpenAI, DataForSEO, Slack, and NocoDB API token for database operations.                                                                                                                                                | Setup as per workflow credentials sections                                                                         |
| Pay attention to API rate limits and quotas for OpenAI and DataForSEO; consider adding error handling or retry nodes if scaling workflow.                                                                                                           | General good practice                                                                                                |
| Structured output parser ensures AI output is reliably parsed; schema changes require corresponding prompt and parser updates.                                                                                                                       | Langchain output parser node configuration                                                                          |
| Slack notifications use a fixed channel ID cached in the node; changing Slack workspace or channel requires updating this ID.                                                                                                                       | Slack node channel configuration                                                                                    |
| NocoDB input and output tables must be created with the exact schema to ensure field mappings and updates work correctly.                                                                                                                           | Tables as described in documentation section                                                                        |
| Potential improvements include adding topic clustering, Google Search Console integration, scheduling, and alerts for keyword metric changes as outlined in the workflow description.                                                              | Section "Possible Improvements"                                                                                      |

---

This comprehensive documentation enables developers and AI agents to fully understand, reproduce, and extend the SEO keyword research workflow with clear guidance on each node's role, configuration, and integration points.