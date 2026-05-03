Research-Based Article Generator with OpenAI and Web Search for High SEO Scores

https://n8nworkflows.xyz/workflows/research-based-article-generator-with-openai-and-web-search-for-high-seo-scores-6820


# Research-Based Article Generator with OpenAI and Web Search for High SEO Scores

### 1. Workflow Overview

This workflow is designed to generate high-SEO research-based articles leveraging OpenAI’s language models and web search capabilities. It targets content creators, SEO specialists, and marketing professionals who want to produce authoritative, well-researched, and structured articles in any desired language.

The workflow logically divides into the following blocks:

- **1.1 Input Reception:** User inputs domain, keywords, and language preferences via a web form.
- **1.2 Keyword Enhancement and Initial Research:** AI refines keywords and performs web search to find related articles and citations.
- **1.3 Article Outline Generation:** Extracts and synthesizes outlines from collected articles and citations.
- **1.4 Section-by-Section Content Creation:** Iteratively researches, analyzes, and writes article sections with SEO optimization and citations.
- **1.5 Article Assembly and Output:** Aggregates all sections into a final Markdown article file ready for export or further refinement.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:** Collects user-provided parameters for the article generation task through a web form.
- **Nodes Involved:**  
  - Form  
  - Language  
  - LLM Params  
  - Sticky Note (general instructions and input context)

- **Node Details:**

  - **Form**  
    - Type: `formTrigger`  
    - Role: Entry point receiving user input: Domain of Expertise, Keywords, Language.  
    - Config: Form fields with user-friendly placeholders and titles.  
    - Connections: Outputs to `Language`.  
    - Potential Failures: Webhook errors, incomplete form submission.

  - **Language**  
    - Type: `set`  
    - Role: Sets internal workflow variables for main language (default English) and output language (from form input).  
    - Config: Assigns `main_language = "English"`, `output_language` dynamically from form.  
    - Connections: Outputs to `LLM Params`.  
    - Potential Failures: Expression errors if form data missing or malformed.

  - **LLM Params**  
    - Type: `set`  
    - Role: Defines critical parameters such as model names (`simple_model`, `advanced_model`), system prompts for various LLM tasks, and working/output languages.  
    - Config:  
      - Models: `gpt-4.1-mini` for simpler tasks, `gpt-4.1` for advanced writing.  
      - System prompts cover keyword generation, article outlines, outline synthesis, section analysis, query generation, search summarization, and section writing.  
      - Language variables consistent with input.  
    - Connections: Outputs to `Generate Keywords`.  
    - Potential Failures: Incorrect prompt syntax, expression interpolation errors.

  - **Sticky Note (Input Context)**  
    - Provides visual documentation on user input role and system prompts usage.

---

#### 2.2 Keyword Enhancement and Initial Research

- **Overview:** Enhances user keywords for SEO, performs web search to find relevant authoritative articles and citations.
- **Nodes Involved:**  
  - Generate Keywords  
  - Collect Keywords  
  - Search Articles  
  - Collect Articles  
  - Analyze Articles  
  - Collect Outlines  
  - Search Citations  
  - Collect Citations  
  - Sticky Notes (explanations for this block)  

- **Node Details:**

  - **Generate Keywords**  
    - Type: `httpRequest` (OpenAI API call)  
    - Role: Refines input keywords using the `simple_model` and a dedicated system prompt for keyword enhancement.  
    - Config: Sends user keywords and working language to OpenAI, expects comma-separated enhanced keywords.  
    - Connections: Outputs to `Collect Keywords`.  
    - Failures: API auth errors, rate limits, malformed JSON, expression errors.

  - **Collect Keywords**  
    - Type: `code`  
    - Role: Parses OpenAI response, extracting the refined keywords text from multiple possible response formats.  
    - Config: JS code handles multiple OpenAI response structures for compatibility.  
    - Connections: Outputs to `Search Articles`.  
    - Failures: Parsing errors if API response format changes.

  - **Search Articles**  
    - Type: `httpRequest` (OpenAI API with web search tool)  
    - Role: Searches for exactly three authoritative articles within the last 12 months on enhanced keywords via OpenAI web search.  
    - Config: Uses `simple_model` with web search preview tool, language and source filters embedded in prompt.  
    - Connections: Outputs to `Collect Articles`.  
    - Failures: Network, API limits, no qualifying articles found.

  - **Collect Articles**  
    - Type: `code`  
    - Role: Extracts ordered Markdown list of articles from OpenAI response.  
    - Connections: Outputs to `Analyze Articles`.  
    - Failures: Parsing errors on response format.

  - **Analyze Articles**  
    - Type: `httpRequest` (OpenAI API call)  
    - Role: Transforms raw article search output into structured outlines using a system prompt for JSON format.  
    - Config: Uses `simple_model` with a JSON schema defined in payload.  
    - Connections: Outputs to `Collect Outlines`.  
    - Failures: API or JSON schema mismatch errors.

  - **Collect Outlines**  
    - Type: `code`  
    - Role: Extracts structured JSON outlines from API response.  
    - Connections: Outputs to `Search Citations`.  
    - Failures: Parsing errors.

  - **Search Citations**  
    - Type: `httpRequest` (OpenAI API with web search tool)  
    - Role: Retrieves exactly five authoritative sources with metadata and one-line takeaways based on enhanced keywords.  
    - Config: Uses `simple_model` with web search, with strict source and recency criteria.  
    - Connections: Outputs to `Collect Citations`.  
    - Failures: API errors, insufficient qualifying sources.

  - **Collect Citations**  
    - Type: `code`  
    - Role: Extracts citations text from API response.  
    - Connections: Outputs to `S1` (start of outline generation).  
    - Failures: Parsing errors.

  - **Sticky Notes**  
    - Document keyword generation, initial research purpose, JavaScript parsing logic, and structured output schema.

---

#### 2.3 Article Outline Generation

- **Overview:** Synthesizes a comprehensive article outline by combining multiple article outlines and citations with SEO keywords.
- **Nodes Involved:**  
  - S1, S2, S3 (noOp for flow control)  
  - Set Outline Prompt  
  - Draft Outline  
  - Collect Outline  
  - S4 (noOp)  
  - Sticky Notes (structured output and schema explanations)

- **Node Details:**

  - **Set Outline Prompt**  
    - Type: `set`  
    - Role: Constructs a detailed prompt combining SEO keywords, three collected outlines, and authoritative citations for advanced model.  
    - Connections: Outputs to `Draft Outline`.  
    - Failures: Expression errors if inputs missing.

  - **Draft Outline**  
    - Type: `httpRequest` (OpenAI API call)  
    - Role: Generates a well-structured, SEO-optimized article outline using `advanced_model` and the prompt from previous node.  
    - Connections: Outputs to `Collect Outline`.  
    - Failures: API errors, timeouts.

  - **Collect Outline**  
    - Type: `code`  
    - Role: Extracts the outline text output from OpenAI response (handling multiple response formats).  
    - Connections: Outputs to `S4`.  
    - Failures: Parsing errors.

  - **S1, S2, S3, S4**  
    - Type: `noOp`  
    - Role: Serve as synchronization or checkpoints in flow control.

  - **Sticky Notes**  
    - Provide explanations of structured output format and JSON schema used for outline generation.

---

#### 2.4 Section-by-Section Content Creation

- **Overview:** Iteratively processes each section of the article outline: analyzes section, generates search queries, performs targeted web searches, summarizes search results, and writes detailed section text.
- **Nodes Involved:**  
  - Separate Sections  
  - Split Sections  
  - Loop Over Items  
  - Analysis Prompt  
  - Analyze Section  
  - Collect Analysis  
  - Generate Queries  
  - Collect Queries  
  - Convert to Array  
  - Split Queries  
  - Prepare Prompt  
  - Web Search  
  - SearchResult  
  - Aggregate  
  - Result Prompt  
  - Search Summary  
  - Collect Summary  
  - Section Prompts  
  - Section Prompt  
  - Write Section  
  - Collect Section  
  - Next Section  
  - Sticky Notes (process explanations and instructions)

- **Node Details:**

  - **Separate Sections**  
    - Type: `set`  
    - Role: Splits article outline text into individual sections by splitting on “\n## ” header markdown.  
    - Connections: Outputs to `Split Sections`.  
    - Failures: Incorrect splitting if outline malformed.

  - **Split Sections**  
    - Type: `splitOut`  
    - Role: Outputs each section as a separate item for batch processing.  
    - Connections: Outputs to `Loop Over Items`.  
    - Failures: None typical.

  - **Loop Over Items**  
    - Type: `splitInBatches`  
    - Role: Processes sections sequentially or in batches, feeding each to section analysis and query generation.  
    - Connections: Outputs to `Aggregate Sections` and `Analysis Prompt`.  
    - Failures: Batch size or item count handling.

  - **Analysis Prompt**  
    - Type: `set`  
    - Role: Prepares user prompt combining current section and full article outline for analysis.  
    - Connections: Outputs to `Analyze Section`.  
    - Failures: Expression errors.

  - **Analyze Section**  
    - Type: `httpRequest` (OpenAI API call)  
    - Role: Analyzes the section structure and content requirements using `simple_model` and a detailed system prompt.  
    - Connections: Outputs to `Collect Analysis`.  
    - Failures: API or network errors.

  - **Collect Analysis**  
    - Type: `code`  
    - Role: Extracts analysis text from OpenAI response.  
    - Connections: Outputs to `Generate Queries`.  
    - Failures: Parsing errors.

  - **Generate Queries**  
    - Type: `httpRequest` (OpenAI API call)  
    - Role: Generates up to 5 specialized web search queries for the section using a dedicated system prompt and `simple_model`.  
    - Connections: Outputs to `Collect Queries`.  
    - Failures: API limits, response format.

  - **Collect Queries**  
    - Type: `code`  
    - Role: Extracts JSON array of queries from response.  
    - Connections: Outputs to `Convert to Array`.  
    - Failures: Parsing errors.

  - **Convert to Array**  
    - Type: `python`  
    - Role: Parses JSON string array into individual items for splitting.  
    - Connections: Outputs to `Split Queries`.  
    - Failures: JSON parsing errors.

  - **Split Queries**  
    - Type: `splitOut`  
    - Role: Splits queries into separate items for sequential web searches.  
    - Connections: Outputs to `Prepare Prompt`.  
    - Failures: None typical.

  - **Prepare Prompt**  
    - Type: `set`  
    - Role: Formats prompt for web search using each query, specifying output format and filters.  
    - Connections: Outputs to `Web Search`.  
    - Failures: Expression errors.

  - **Web Search**  
    - Type: `httpRequest` (OpenAI API call with web search tool)  
    - Role: Runs each query against OpenAI web search to retrieve raw content previews.  
    - Connections: Outputs to `SearchResult`.  
    - Failures: API failures, no results.

  - **SearchResult**  
    - Type: `set`  
    - Role: Formats search results into a uniform object with query and text content.  
    - Connections: Outputs to `Aggregate`.  
    - Failures: None typical.

  - **Aggregate**  
    - Type: `aggregate`  
    - Role: Collects all search results for a section into a single array.  
    - Connections: Outputs to `Result Prompt`.  
    - Failures: None typical.

  - **Result Prompt**  
    - Type: `set`  
    - Role: Prepares input for search summary with aggregated search results.  
    - Connections: Outputs to `Search Summary`.  
    - Failures: Expression errors.

  - **Search Summary**  
    - Type: `httpRequest` (OpenAI API call)  
    - Role: Synthesizes raw search results into a coherent research summary using `advanced_model`.  
    - Connections: Outputs to `Collect Summary`.  
    - Failures: API errors.

  - **Collect Summary**  
    - Type: `code`  
    - Role: Extracts summary text from response.  
    - Connections: Outputs to `Section Prompt`.  
    - Failures: Parsing errors.

  - **Section Prompts**  
    - Type: `set`  
    - Role: Holds system prompts for section content writing, including analysis, query generation, and writing instructions.  
    - Connections: Outputs to `Section Prompt`.  
    - Failures: None typical.

  - **Section Prompt**  
    - Type: `set`  
    - Role: Builds user prompt combining section outline, SEO keywords, search summary, and citations for writing.  
    - Connections: Outputs to `Write Section`.  
    - Failures: Expression errors.

  - **Write Section**  
    - Type: `httpRequest` (OpenAI API call)  
    - Role: Generates the article section text in target language using `advanced_model` with rich content and citations.  
    - Connections: Outputs to `Collect Section`.  
    - Failures: API timeouts, formatting errors.

  - **Collect Section**  
    - Type: `code`  
    - Role: Extracts final section text from response.  
    - Connections: Outputs to `Next Section`.  
    - Failures: Parsing errors.

  - **Next Section**  
    - Type: `noOp`  
    - Role: Marks end of current section processing, triggers next batch iteration.  
    - Connections: Loops back to `Loop Over Items`.  
    - Failures: None.

  - **Sticky Notes**  
    - Explain iterative section writing, research, analysis, and finalization. Notes on reflection and human-in-the-loop improvements.

---

#### 2.5 Article Assembly and Output

- **Overview:** Aggregates all written sections, formats the article title, exports finalized Markdown file.
- **Nodes Involved:**  
  - Aggregate Sections  
  - Combine Article  
  - Markdown File  
  - Sticky Note (article ready notice)  

- **Node Details:**

  - **Aggregate Sections**  
    - Type: `aggregate`  
    - Role: Collects all section texts from batch iterations into a single data array.  
    - Connections: Outputs to `Combine Article`.  
    - Failures: None typical.

  - **Combine Article**  
    - Type: `set`  
    - Role:  
      - Extracts article title from first outline header line.  
      - Joins all section texts with double newlines for Markdown formatting.  
    - Connections: Outputs to `Markdown File`.  
    - Failures: Expression errors if outline malformed.

  - **Markdown File**  
    - Type: `convertToFile`  
    - Role: Generates a .md text file from combined article content, filename derived from title.  
    - Connections: No downstream nodes.  
    - Failures: File system or export errors.

  - **Sticky Note (Article Ready)**  
    - Indicates article is ready in Markdown, suggests next steps like polish, SEO check, export.

---

### 3. Summary Table

| Node Name           | Node Type                  | Functional Role                                  | Input Node(s)       | Output Node(s)           | Sticky Note                                                                                                  |
|---------------------|----------------------------|------------------------------------------------|---------------------|--------------------------|--------------------------------------------------------------------------------------------------------------|
| Form                | formTrigger                | User input collection                           |                     | Language                 |                                                                                                              |
| Language            | set                        | Sets main and output language variables         | Form                | LLM Params               |                                                                                                              |
| LLM Params          | set                        | Defines model names, system prompts, languages  | Language             | Generate Keywords         | Model params and system prompts details                                                                       |
| Generate Keywords    | httpRequest (OpenAI)        | Enhances keywords via AI                         | LLM Params           | Collect Keywords          |                                                                                                              |
| Collect Keywords     | code                       | Parses AI keyword response                       | Generate Keywords    | Search Articles           | JS code handles multiple response formats                                                                    |
| Search Articles      | httpRequest (OpenAI + web)  | Retrieves 3 authoritative articles              | Collect Keywords     | Collect Articles          |                                                                                                              |
| Collect Articles     | code                       | Extracts article list from AI response          | Search Articles      | Analyze Articles          |                                                                                                              |
| Analyze Articles     | httpRequest (OpenAI)        | Converts articles info into structured outlines | Collect Articles     | Collect Outlines          | Structured output format                                                                                        |
| Collect Outlines     | code                       | Extracts outlines JSON                           | Analyze Articles     | Search Citations          |                                                                                                              |
| Search Citations     | httpRequest (OpenAI + web)  | Retrieves 5 authoritative citations             | Collect Outlines     | Collect Citations         |                                                                                                              |
| Collect Citations    | code                       | Parses citations response                        | Search Citations     | S1                       |                                                                                                              |
| S1                  | noOp                       | Flow control checkpoint                         | Collect Citations    | S2                       |                                                                                                              |
| S2                  | noOp                       | Flow control checkpoint                         | S1                   | S3                       |                                                                                                              |
| S3                  | noOp                       | Flow control checkpoint                         | S2                   | Set Outline Prompt        |                                                                                                              |
| Set Outline Prompt   | set                        | Builds combined prompt for outline generation   | S3                   | Draft Outline             |                                                                                                              |
| Draft Outline        | httpRequest (OpenAI)        | Generates full article outline                   | Set Outline Prompt   | Collect Outline           |                                                                                                              |
| Collect Outline      | code                       | Extracts outline text                            | Draft Outline        | S4                       |                                                                                                              |
| S4                  | noOp                       | Flow control checkpoint                         | Collect Outline      | Separate Sections         |                                                                                                              |
| Separate Sections    | set                        | Splits outline into sections                     | S4                   | Split Sections            |                                                                                                              |
| Split Sections       | splitOut                   | Outputs individual sections                      | Separate Sections    | Loop Over Items           |                                                                                                              |
| Loop Over Items      | splitInBatches             | Processes each section iteratively               | Split Sections       | Aggregate Sections, Analysis Prompt |                                                                                                              |
| Analysis Prompt      | set                        | Prepares prompt for section analysis             | Loop Over Items      | Analyze Section           |                                                                                                              |
| Analyze Section      | httpRequest (OpenAI)        | Analyzes section content and structure           | Analysis Prompt      | Collect Analysis          |                                                                                                              |
| Collect Analysis     | code                       | Extracts analysis result                         | Analyze Section      | Generate Queries          |                                                                                                              |
| Generate Queries     | httpRequest (OpenAI)        | Creates search queries for section                | Collect Analysis     | Collect Queries           |                                                                                                              |
| Collect Queries      | code                       | Parses query list JSON                           | Generate Queries     | Convert to Array          |                                                                                                              |
| Convert to Array     | python                     | Converts JSON string to array items              | Collect Queries      | Split Queries             |                                                                                                              |
| Split Queries        | splitOut                   | Outputs individual queries for search            | Convert to Array     | Prepare Prompt            |                                                                                                              |
| Prepare Prompt       | set                        | Formats web search prompts                        | Split Queries        | Web Search                |                                                                                                              |
| Web Search           | httpRequest (OpenAI + web)  | Executes web search for each query                | Prepare Prompt       | SearchResult              |                                                                                                              |
| SearchResult         | set                        | Formats search results                            | Web Search           | Aggregate                 |                                                                                                              |
| Aggregate            | aggregate                  | Collects all search results for a section         | SearchResult         | Result Prompt             |                                                                                                              |
| Result Prompt        | set                        | Prepares aggregated search results for summary   | Aggregate            | Search Summary            |                                                                                                              |
| Search Summary       | httpRequest (OpenAI)        | Summarizes raw search results                      | Result Prompt        | Collect Summary           |                                                                                                              |
| Collect Summary      | code                       | Extracts summary text                             | Search Summary       | Section Prompt            |                                                                                                              |
| Section Prompts      | set                        | Contains system prompts for section writing       | Loop Over Items      | Section Prompt            |                                                                                                              |
| Section Prompt       | set                        | Builds prompt for writing section content         | Collect Summary      | Write Section             |                                                                                                              |
| Write Section        | httpRequest (OpenAI)        | Generates section text                             | Section Prompt       | Collect Section           |                                                                                                              |
| Collect Section      | code                       | Extracts section text                             | Write Section        | Next Section              |                                                                                                              |
| Next Section         | noOp                       | Triggers next section processing                   | Collect Section      | Loop Over Items           |                                                                                                              |
| Aggregate Sections   | aggregate                  | Aggregates all sections' texts                      | Loop Over Items      | Combine Article           |                                                                                                              |
| Combine Article      | set                        | Forms article title and joins sections             | Aggregate Sections   | Markdown File             |                                                                                                              |
| Markdown File        | convertToFile              | Exports final article as Markdown file              | Combine Article      |                          |                                                                                                              |
| Sticky Notes         | stickyNote                 | Documentation and explanations                      | Various              |                          | Multiple notes explain input, keyword generation, structured output, section writing, article readiness, and usage instructions. |

---

### 4. Reproducing the Workflow from Scratch

**Step 1: Create the Input Form Node**

- Node Type: `formTrigger`  
- Configure form fields:  
  - Domain of Expertize (text input)  
  - Keywords (text input)  
  - Language (text input)  
- Set form title and description accordingly.  
- Save the webhook URL for access.

**Step 2: Create “Language” Set Node**

- Node Type: `set`  
- Assign:  
  - `main_language` = `"English"` (default)  
  - `output_language` = expression referencing form input `Language` (`={{ $json.Language }}`)  
- Connect output from Form to this node.

**Step 3: Create “LLM Params” Set Node**

- Node Type: `set`  
- Assign:  
  - `simple_model` = `"gpt-4.1-mini"`  
  - `advanced_model` = `"gpt-4.1"`  
  - `working_language` = `"English"`  
  - `output_language` = `={{ $json.output_language }}` (from Language node)  
  - Several system prompt variables as multiline strings (copy from original, adapt placeholders):  
    - `system_prompt_generate_kw`  
    - `system_prompt_article_outlines`  
    - `system_prompt_outline`  
    - `system_prompt_search_summary`  
    - `system_prompt_write_section`  
    - `system_prompt_generate_queries`  
    - `sytem_prompt_analyze_section` (note spelling)  
- Connect output from Language node.

**Step 4: Create “Generate Keywords” HTTP Request Node**

- Type: `httpRequest`  
- Method: POST  
- URL: `https://api.openai.com/v1/responses`  
- Authentication: OpenAI API credentials required  
- Body (JSON):  
  - Use model from `LLM Params.simple_model`  
  - Input: system prompt for keyword generation + user keywords from form + working language  
- Connect output from `LLM Params`.

**Step 5: Create “Collect Keywords” Code Node**

- Node Type: `code` (JavaScript)  
- Logic: parse OpenAI response to extract keyword text regardless of response format.  
- Connect output from `Generate Keywords`.

**Step 6: Create “Search Articles” HTTP Request Node**

- Type: `httpRequest`  
- Method: POST  
- Use OpenAI API with web search preview tool enabled  
- Model: `simple_model` from `LLM Params`  
- Input: prompt requesting exactly 3 authoritative articles published within last 12 months on keywords from `Collect Keywords` output  
- Connect output from `Collect Keywords`.

**Step 7: Create “Collect Articles” Code Node**

- Parses search response extracting markdown list of article summaries.  
- Connect output from `Search Articles`.

**Step 8: Create “Analyze Articles” HTTP Request Node**

- Uses `simple_model` with prompt to extract and reformat article data into JSON outlines per schema defined.  
- Connect output from `Collect Articles`.

**Step 9: Create “Collect Outlines” Code Node**

- Extracts JSON structured outlines from API response.  
- Connect output from `Analyze Articles`.

**Step 10: Create “Search Citations” HTTP Request Node**

- Similar config as `Search Articles` but searches for 5 authoritative citations with metadata and one-line takeaways.  
- Uses keywords output from `Collect Keywords`.  
- Connect from `Collect Outlines`.

**Step 11: Create “Collect Citations” Code Node**

- Extracts citation list from response.  
- Connect output from `Search Citations`.

**Step 12: Create Flow Control Nodes (noOp) S1, S2, S3**

- Use `noOp` nodes to manage sequential steps.  
- Connect flow: `Collect Citations` → S1 → S2 → S3.

**Step 13: Create “Set Outline Prompt” Set Node**

- Builds a user prompt combining:  
  - SEO keywords from form  
  - Outlines from `Collect Outlines` node (parse JSON)  
  - Citations from `Collect Citations`  
- Connect output from S3.

**Step 14: Create “Draft Outline” HTTP Request Node**

- Calls OpenAI with `advanced_model` using prompt from “Set Outline Prompt”.  
- Generates a full, SEO-optimized article outline.  
- Connect output from “Set Outline Prompt”.

**Step 15: Create “Collect Outline” Code Node**

- Parses outline text output.  
- Connect output from “Draft Outline”.

**Step 16: Create “Separate Sections” Set Node**

- Splits outline text by section headers (`\n## `) into array `sections`.  
- Connect output from “Collect Outline”.

**Step 17: Create “Split Sections” splitOut Node**

- Splits array `sections` into individual items for batch processing.  
- Connect output from “Separate Sections”.

**Step 18: Create “Loop Over Items” splitInBatches Node**

- Processes each section sequentially for research and writing.  
- Connect output from “Split Sections”.

**Step 19: Create “Analysis Prompt” Set Node**

- Constructs prompt for section analysis combining current section text and full article outline.  
- Connect output from “Loop Over Items”.

**Step 20: Create “Analyze Section” HTTP Request Node**

- Uses `simple_model` with system prompt for section analysis.  
- Connect output from “Analysis Prompt”.

**Step 21: Create “Collect Analysis” Code Node**

- Extracts analysis text.  
- Connect output from “Analyze Section”.

**Step 22: Create “Generate Queries” HTTP Request Node**

- Creates up to 5 web search queries for each section.  
- Connect output from “Collect Analysis”.

**Step 23: Create “Collect Queries” Code Node**

- Parses JSON array of queries.  
- Connect output from “Generate Queries”.

**Step 24: Create “Convert to Array” Python Node**

- Parses JSON string array into individual items.  
- Connect output from “Collect Queries”.

**Step 25: Create “Split Queries” splitOut Node**

- Splits queries into individual items for search.  
- Connect output from “Convert to Array”.

**Step 26: Create “Prepare Prompt” Set Node**

- Formats web search prompt using each query.  
- Connect output from “Split Queries”.

**Step 27: Create “Web Search” HTTP Request Node**

- Calls OpenAI with web search tool using prepared prompt.  
- Connect output from “Prepare Prompt”.

**Step 28: Create “SearchResult” Set Node**

- Formats search result with query and content.  
- Connect output from “Web Search”.

**Step 29: Create “Aggregate” Node**

- Aggregates search results for the section.  
- Connect output from “SearchResult”.

**Step 30: Create “Result Prompt” Set Node**

- Prepares aggregated results for summarization.  
- Connect output from “Aggregate”.

**Step 31: Create “Search Summary” HTTP Request Node**

- Summarizes aggregated search results with `advanced_model`.  
- Connect output from “Result Prompt”.

**Step 32: Create “Collect Summary” Code Node**

- Extracts summary text.  
- Connect output from “Search Summary”.

**Step 33: Create “Section Prompts” Set Node**

- Stores system prompts for section writing and other subtasks.  
- Connect output from “Loop Over Items”.

**Step 34: Create “Section Prompt” Set Node**

- Builds final user prompt for section writing combining outline, keywords, summary, citations.  
- Connect output from “Collect Summary”.

**Step 35: Create “Write Section” HTTP Request Node**

- Calls OpenAI `advanced_model` to generate section content in target language.  
- Connect output from “Section Prompt”.

**Step 36: Create “Collect Section” Code Node**

- Extracts section text.  
- Connect output from “Write Section”.

**Step 37: Create “Next Section” noOp Node**

- Marks end of section processing, loops back to “Loop Over Items”.  
- Connect output from “Collect Section” to “Next Section” and from “Next Section” back to “Loop Over Items”.

**Step 38: Create “Aggregate Sections” Node**

- Aggregates all section texts after batch processing completes.  
- Connect output from “Loop Over Items”.

**Step 39: Create “Combine Article” Set Node**

- Extracts article title from outline and joins all sections into a single Markdown string.  
- Connect output from “Aggregate Sections”.

**Step 40: Create “Markdown File” Convert to File Node**

- Converts combined article text to `.md` file. Filename is the article title.  
- Connect output from “Combine Article”.

**Step 41: Add Sticky Notes**

- Create sticky notes throughout the workflow to explain input expectations, model parameters, prompt instructions, workflow logic, and next steps (e.g., SEO polishing, export).

---

### 5. General Notes & Resources

| Note Content                                                                           | Context or Link                                                                                                  |
|----------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------|
| Enter your OpenAI API credentials and set `simple_model` and `advanced_model` names in the LLM Params node. | Quickstart instructions (Sticky Note).                                                                         |
| Use this workflow as a cornerstone AI content assistant for research-based SEO articles. | Workflow description and purpose (Sticky Note).                                                                |
| JavaScript code in "Collect ..." nodes handles multiple OpenAI API response formats for forward compatibility. | Code robustness explanation (Sticky Notes).                                                                     |
| Structured JSON schema used for article outlines to ensure consistent parsing and generation. | Object schema documentation (Sticky Notes).                                                                     |
| Section writing is iterative: research, query generation, web search, summary, writing, and aggregation per section. | Process explanation (Sticky Notes).                                                                              |
| Final article is exported as Markdown, ready for further polishing, SEO checks, or format conversion (HTML, PDF). | Article readiness note (Sticky Note).                                                                            |
| Consider adding human-in-the-loop review or AI reflection on generated sections for improved quality. | Suggestion for workflow enhancement (Sticky Note).                                                              |
| Workflow uses OpenAI's web_search_preview tool for real-time web content integration in research. | Integration detail.                                                                                               |
| Remove any UTM or tracking parameters from URLs in citations and articles for clean source referencing. | Prompt instructions in citation search and outline generation nodes.                                            |
| Use Markdown (ATX) formatting consistently for outlines and article sections for compatibility with common markdown processors. | Formatting guideline.                                                                                            |

---

**Disclaimer:**  
The text described and processed in this documentation originates exclusively from an automated n8n workflow built for legitimate content generation purposes. It complies with all applicable content policies and uses only legal and public data sources.