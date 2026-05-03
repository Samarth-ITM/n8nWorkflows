Customer Feedback Analysis with AI, QuickChart & HTML Report Generator

https://n8nworkflows.xyz/workflows/customer-feedback-analysis-with-ai--quickchart---html-report-generator-3642


# Customer Feedback Analysis with AI, QuickChart & HTML Report Generator

### 1. Workflow Overview

This workflow, titled **Customer Feedback Analysis with AI, QuickChart & HTML Report Generator**, automates the semantic analysis of customer feedback or social media data stored in Google Sheets. It leverages AI agents (via LangChain nodes and OpenAI/DeepSeek LLMs) to generate insightful prompts, analyze feedback row-by-row, refine analysis through iterative AI processing, and produce visualizations and a comprehensive HTML report. The final report is automatically emailed via Gmail.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Preprocessing**  
  Import raw feedback data from Google Sheets, filter sample rows for prompt generation.

- **1.2 Prompt Proposal Generation**  
  Use AI to generate generalizable prompts for analyzing feedback data.

- **1.3 Prompt Injection and Pairing**  
  Combine each feedback row with all generated prompts to prepare for row-level analysis.

- **1.4 First Iteration of Row-Level Analysis**  
  AI evaluates each row against all prompts, producing structured outputs.

- **1.5 Semantic Merging and Prompt Refinement**  
  Merge outputs, cluster synonyms, and refine prompts using AI.

- **1.6 Second Iteration of Analysis**  
  Re-analyze rows with refined prompts to produce improved structured data.

- **1.7 Summarization and Visualization**  
  Generate dimension-wise summaries, create QuickChart visualizations, and produce cross-dimensional insights.

- **1.8 Final Report Generation and Email Delivery**  
  Format the final HTML report and send it via Gmail.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Preprocessing

- **Overview:**  
  This block retrieves customer feedback data from a Google Sheet, filters the first 20 rows to create a sample dataset for prompt generation, and merges data for further processing.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Google Sheets  
  - Get the first n rows (Filter)  
  - Merging first sample rows (Code)  
  - Merge original table and the prompts (Merge)

- **Node Details:**

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point to start the workflow manually.  
    - Inputs: None  
    - Outputs: Triggers Google Sheets node.  
    - Edge cases: None.

  - **Google Sheets**  
    - Type: Google Sheets node  
    - Role: Fetches raw feedback data from a configured Google Sheet tab.  
    - Configuration: Uses preconfigured Google Sheets credentials; sheet ID and tab are set to target feedback data.  
    - Inputs: Trigger from manual node.  
    - Outputs: Data passed to filter and merge nodes.  
    - Edge cases: Authentication errors, sheet access issues, empty or malformed data.

  - **Get the first n rows**  
    - Type: Filter node  
    - Role: Filters the first 20 rows from the Google Sheets data for prompt proposal generation.  
    - Configuration: Filter condition set to limit rows to 20.  
    - Inputs: Data from Google Sheets.  
    - Outputs: Filtered rows to Merging first sample rows node.  
    - Edge cases: Less than 20 rows available.

  - **Merging first sample rows**  
    - Type: Code node  
    - Role: Prepares and merges the filtered sample rows into a format suitable for prompt generation.  
    - Configuration: Custom JavaScript code to aggregate rows.  
    - Inputs: Filtered rows.  
    - Outputs: Merged sample data to analysis topics proposal node.  
    - Edge cases: Code errors, empty input.

  - **Merge original table and the prompts**  
    - Type: Merge node  
    - Role: Combines original feedback data with generated prompts for row-level pairing.  
    - Configuration: Merge mode set to combine datasets appropriately (likely 'append' or 'inner join').  
    - Inputs: Original Google Sheets data and prompts from AI agent.  
    - Outputs: Combined dataset for analysis.  
    - Edge cases: Mismatched data structures.

---

#### 1.2 Prompt Proposal Generation

- **Overview:**  
  AI generates 3 to 6 generalizable prompts for analyzing feedback rows, structured in JSON and agnostic to specific product names or column headers.

- **Nodes Involved:**  
  - analysis topics proposal (LangChain Agent)  
  - OpenAI Chat Model1 (LLM)  
  - Structured Output Parser1 (LangChain Output Parser)

- **Node Details:**

  - **analysis topics proposal**  
    - Type: LangChain Agent  
    - Role: Generates initial analysis prompts based on sample data.  
    - Configuration: Uses an AI model (OpenAI or DeepSeek) with instructions to produce JSON-formatted prompts.  
    - Inputs: Merged sample rows.  
    - Outputs: Raw AI-generated prompts.  
    - Edge cases: AI output formatting errors, API timeouts.

  - **OpenAI Chat Model1**  
    - Type: LangChain LLM Chat Model  
    - Role: Provides the language model backend for the agent.  
    - Configuration: Model parameters set for chat completion; uses configured OpenAI or DeepSeek credentials.  
    - Inputs: Prompts from agent node.  
    - Outputs: AI-generated prompt content.  
    - Edge cases: API quota limits, authentication errors.

  - **Structured Output Parser1**  
    - Type: LangChain Output Parser  
    - Role: Parses AI output into structured JSON for downstream processing.  
    - Configuration: Parser expects JSON format matching prompt schema.  
    - Inputs: Raw AI prompt output.  
    - Outputs: Parsed prompt objects.  
    - Edge cases: Parsing failures if AI output is malformed.

---

#### 1.3 Prompt Injection and Pairing

- **Overview:**  
  Each feedback row is paired with all generated prompts, creating a combined dataset for row-level AI evaluation.

- **Nodes Involved:**  
  - Merge original table and the prompts (Merge)  
  - Prepare Prompts1 (Function)  
  - Unified AI Agent for analysis (LangChain Agent)  

- **Node Details:**

  - **Prepare Prompts1**  
    - Type: Function node  
    - Role: Prepares and formats the combined dataset of rows and prompts for AI analysis.  
    - Configuration: Custom JavaScript to inject prompts into each row.  
    - Inputs: Merged data from previous merge node.  
    - Outputs: Prepared dataset for AI agent.  
    - Edge cases: Data structure mismatches.

  - **Unified AI Agent for analysis**  
    - Type: LangChain Agent  
    - Role: Performs row-level analysis using the combined prompts and feedback data.  
    - Configuration: Uses OpenAI Chat Model as backend; configured to produce structured analysis per prompt per row.  
    - Inputs: Prepared dataset.  
    - Outputs: Raw AI analysis results.  
    - Edge cases: API errors, large input size causing timeouts.

  - **OpenAI Chat Model**  
    - Type: LangChain LLM Chat Model  
    - Role: Provides language model for the Unified AI Agent.  
    - Configuration: Model parameters tuned for analysis tasks.  
    - Inputs: Prompts from agent.  
    - Outputs: AI-generated analysis.  
    - Edge cases: Same as above.

  - **Structured Output Parser**  
    - Type: LangChain Output Parser  
    - Role: Parses AI analysis output into structured fields.  
    - Inputs: Raw AI output.  
    - Outputs: Structured analysis data.  
    - Edge cases: Parsing errors.

---

#### 1.4 First Iteration of Row-Level Analysis

- **Overview:**  
  AI evaluates each row against all prompts, producing structured outputs that are transformed and merged for refinement.

- **Nodes Involved:**  
  - Transform results into columns (Code)  
  - All unique elements merge (Code)  
  - proposal refinement agent (LangChain Agent)  
  - OpenAI Chat Model2 (LLM)  
  - Structured Output Parser2 (LangChain Output Parser)  
  - Merge (Merge node)

- **Node Details:**

  - **Transform results into columns**  
    - Type: Code node  
    - Role: Converts AI output into columnar format for easier merging.  
    - Configuration: Custom JavaScript to reshape data.  
    - Inputs: Structured AI analysis data.  
    - Outputs: Reformatted data.  
    - Edge cases: Data inconsistency.

  - **All unique elements merge**  
    - Type: Code node  
    - Role: Aggregates unique values from analysis results to prepare for prompt refinement.  
    - Configuration: Custom code to deduplicate and merge lists.  
    - Inputs: Reformatted data.  
    - Outputs: Aggregated unique elements.  
    - Edge cases: Empty or malformed input.

  - **proposal refinement agent**  
    - Type: LangChain Agent  
    - Role: Uses AI to cluster synonyms and regenerate improved prompt definitions.  
    - Configuration: Connected to OpenAI Chat Model2; expects structured input and outputs refined prompts.  
    - Inputs: Aggregated unique elements.  
    - Outputs: Refined prompt definitions.  
    - Edge cases: AI output format errors.

  - **OpenAI Chat Model2**  
    - Type: LangChain LLM Chat Model  
    - Role: Backend for proposal refinement agent.  
    - Inputs: Prompt refinement instructions.  
    - Outputs: Refined prompt JSON.  
    - Edge cases: API limits.

  - **Structured Output Parser2**  
    - Type: LangChain Output Parser  
    - Role: Parses refined prompt output into structured JSON.  
    - Inputs: Raw AI output.  
    - Outputs: Parsed refined prompts.  
    - Edge cases: Parsing failures.

  - **Merge**  
    - Type: Merge node  
    - Role: Combines refined prompts with original data for the next analysis iteration.  
    - Inputs: Refined prompts and original data.  
    - Outputs: Dataset for second iteration.  
    - Edge cases: Data mismatch.

---

#### 1.5 Second Iteration of Analysis

- **Overview:**  
  The refined prompts are used to re-analyze each feedback row, producing improved structured outputs that are wrapped into a single JSON object.

- **Nodes Involved:**  
  - Second iteration of analysis (LangChain Agent)  
  - OpenAI Chat Model3 (LLM)  
  - Structured Output Parser3 (LangChain Output Parser)  
  - wrap up the whole results into one Json file (Code)

- **Node Details:**

  - **Second iteration of analysis**  
    - Type: LangChain Agent  
    - Role: Performs refined row-level analysis using improved prompts.  
    - Configuration: Uses OpenAI Chat Model3; expects merged data.  
    - Inputs: Merged refined prompts and feedback rows.  
    - Outputs: Refined AI analysis results.  
    - Edge cases: API errors, large input size.

  - **OpenAI Chat Model3**  
    - Type: LangChain LLM Chat Model  
    - Role: Backend for second iteration agent.  
    - Inputs: Refined prompts and data.  
    - Outputs: AI analysis.  
    - Edge cases: Same as above.

  - **Structured Output Parser3**  
    - Type: LangChain Output Parser  
    - Role: Parses second iteration AI output into structured JSON.  
    - Inputs: Raw AI output.  
    - Outputs: Parsed refined analysis.  
    - Edge cases: Parsing errors.

  - **wrap up the whole results into one Json file**  
    - Type: Code node  
    - Role: Aggregates all row-level refined analysis into a single JSON object for summarization.  
    - Configuration: Custom JavaScript code.  
    - Inputs: Parsed refined analysis.  
    - Outputs: Wrapped JSON for summarization.  
    - Edge cases: Code errors, empty input.

---

#### 1.6 Summarization and Visualization

- **Overview:**  
  AI generates dimension-wise summaries, creates QuickChart visualizations, and produces cross-dimensional insights and a global narrative.

- **Nodes Involved:**  
  - Summarization of the unalysed results (LangChain Agent)  
  - OpenAI Chat Model4 (LLM)  
  - Structured Output Parser4 (LangChain Output Parser)  
  - OpenAI Chat Model5 (LLM)  
  - Final report editor (LangChain Agent)  
  - Structured Output Parser5 (LangChain Output Parser)  
  - Revisor and HTML formating agent (LangChain Agent)  
  - OpenAI Chat Model6 (LLM)

- **Node Details:**

  - **Summarization of the unalysed results**  
    - Type: LangChain Agent  
    - Role: Generates summaries per analysis dimension and prepares visualization data.  
    - Configuration: Connected to OpenAI Chat Model4.  
    - Inputs: Wrapped JSON analysis results.  
    - Outputs: Summaries and visualization instructions.  
    - Edge cases: AI output format errors.

  - **OpenAI Chat Model4**  
    - Type: LangChain LLM Chat Model  
    - Role: Backend for summarization agent.  
    - Inputs: Summarization prompts.  
    - Outputs: Summaries.  
    - Edge cases: API limits.

  - **Structured Output Parser4**  
    - Type: LangChain Output Parser  
    - Role: Parses summarization output into structured data.  
    - Inputs: Raw AI output.  
    - Outputs: Parsed summaries.  
    - Edge cases: Parsing failures.

  - **OpenAI Chat Model5**  
    - Type: LangChain LLM Chat Model  
    - Role: Supports final report editing.  
    - Inputs: Summaries and visualization data.  
    - Outputs: Edited report content.  
    - Edge cases: API errors.

  - **Final report editor**  
    - Type: LangChain Agent  
    - Role: Produces the final report draft including charts and narratives.  
    - Inputs: Parsed summaries.  
    - Outputs: Draft report content.  
    - Edge cases: AI output formatting.

  - **Structured Output Parser5**  
    - Type: LangChain Output Parser  
    - Role: Parses final report draft into structured format.  
    - Inputs: Raw AI output.  
    - Outputs: Parsed report content.  
    - Edge cases: Parsing errors.

  - **Revisor and HTML formating agent**  
    - Type: LangChain Agent  
    - Role: Revises and formats the report into HTML.  
    - Inputs: Parsed report content.  
    - Outputs: Final HTML report.  
    - Edge cases: Formatting errors.

  - **OpenAI Chat Model6**  
    - Type: LangChain LLM Chat Model  
    - Role: Backend for revising and formatting agent.  
    - Inputs: Report draft.  
    - Outputs: HTML formatted report.  
    - Edge cases: API errors.

---

#### 1.7 Final Report Generation and Email Delivery

- **Overview:**  
  The finalized HTML report is optionally post-processed and sent via Gmail.

- **Nodes Involved:**  
  - Completion agents (optional) (LangChain Agent)  
  - OpenAI Chat Model7 (LLM)  
  - Edit Fields (Set node)  
  - Gmail

- **Node Details:**

  - **Completion agents (optional)**  
    - Type: LangChain Agent  
    - Role: Optional final touches or additional completions on the report content.  
    - Inputs: HTML report.  
    - Outputs: Enhanced report content.  
    - Edge cases: Optional node; may be skipped.

  - **OpenAI Chat Model7**  
    - Type: LangChain LLM Chat Model  
    - Role: Backend for completion agents.  
    - Inputs: Report content.  
    - Outputs: Completed report.  
    - Edge cases: API limits.

  - **Edit Fields**  
    - Type: Set node  
    - Role: Prepares email fields such as recipient, subject, and body with the final report.  
    - Inputs: Completed report content.  
    - Outputs: Email-ready data.  
    - Edge cases: Missing email parameters.

  - **Gmail**  
    - Type: Gmail node  
    - Role: Sends the final HTML report via email using OAuth2 credentials.  
    - Configuration: Uses Gmail OAuth2 credentials; email parameters set from previous node.  
    - Inputs: Email data.  
    - Outputs: Email sent confirmation.  
    - Edge cases: Authentication errors, quota limits, invalid email addresses.

---

### 3. Summary Table

| Node Name                         | Node Type                          | Functional Role                              | Input Node(s)                      | Output Node(s)                      | Sticky Note                                                                                  |
|----------------------------------|----------------------------------|----------------------------------------------|----------------------------------|-----------------------------------|----------------------------------------------------------------------------------------------|
| When clicking ‘Test workflow’     | Manual Trigger                   | Entry point to start the workflow             | None                             | Google Sheets                     |                                                                                              |
| Google Sheets                    | Google Sheets                    | Fetches raw feedback data                      | When clicking ‘Test workflow’    | Get the first n rows, Merge nodes |                                                                                              |
| Get the first n rows             | Filter                          | Filters first 20 rows for prompt generation   | Google Sheets                   | Merging first sample rows          |                                                                                              |
| Merging first sample rows        | Code                            | Aggregates sample rows for prompt proposal    | Get the first n rows             | analysis topics proposal           |                                                                                              |
| analysis topics proposal         | LangChain Agent                 | Generates initial analysis prompts             | Merging first sample rows        | Merge original table and the prompts |                                                                                              |
| OpenAI Chat Model1               | LangChain LLM Chat Model        | LLM backend for prompt proposal agent         | analysis topics proposal         | analysis topics proposal           |                                                                                              |
| Structured Output Parser1        | LangChain Output Parser         | Parses prompt proposal output                   | analysis topics proposal         | Merge original table and the prompts |                                                                                              |
| Merge original table and the prompts | Merge                          | Combines original data with prompts            | Google Sheets, analysis topics proposal | Prepare Prompts1                 |                                                                                              |
| Prepare Prompts1                | Function                        | Injects prompts into each row                   | Merge original table and the prompts | Unified AI Agent for analysis    |                                                                                              |
| Unified AI Agent for analysis    | LangChain Agent                 | Performs row-level analysis                      | Prepare Prompts1                | Transform results into columns     |                                                                                              |
| OpenAI Chat Model               | LangChain LLM Chat Model        | LLM backend for unified AI agent                | Unified AI Agent for analysis    | Unified AI Agent for analysis      |                                                                                              |
| Structured Output Parser        | LangChain Output Parser         | Parses row-level AI analysis output              | Unified AI Agent for analysis    | Transform results into columns     |                                                                                              |
| Transform results into columns  | Code                            | Converts AI output into columnar format          | Structured Output Parser         | All unique elements merge          |                                                                                              |
| All unique elements merge       | Code                            | Aggregates unique analysis elements              | Transform results into columns   | proposal refinement agent          |                                                                                              |
| proposal refinement agent       | LangChain Agent                 | Refines prompts by clustering synonyms          | All unique elements merge        | Merge                            |                                                                                              |
| OpenAI Chat Model2              | LangChain LLM Chat Model        | LLM backend for proposal refinement              | proposal refinement agent        | proposal refinement agent          |                                                                                              |
| Structured Output Parser2       | LangChain Output Parser         | Parses refined prompt output                      | proposal refinement agent        | Merge                            |                                                                                              |
| Merge                         | Merge                          | Combines refined prompts with original data      | proposal refinement agent, Google Sheets | Second iteration of analysis     |                                                                                              |
| Second iteration of analysis    | LangChain Agent                 | Re-analyzes rows with refined prompts            | Merge                           | wrap up the whole results into one Json file |                                                                                              |
| OpenAI Chat Model3             | LangChain LLM Chat Model        | LLM backend for second iteration analysis         | Second iteration of analysis     | Second iteration of analysis       |                                                                                              |
| Structured Output Parser3      | LangChain Output Parser         | Parses second iteration AI output                  | Second iteration of analysis     | wrap up the whole results into one Json file |                                                                                              |
| wrap up the whole results into one Json file | Code                            | Aggregates all refined analysis into one JSON     | Structured Output Parser3        | Summarization of the unalysed results |                                                                                              |
| Summarization of the unalysed results | LangChain Agent                 | Generates summaries and visualization data        | wrap up the whole results into one Json file | Final report editor             |                                                                                              |
| OpenAI Chat Model4             | LangChain LLM Chat Model        | LLM backend for summarization                      | Summarization of the unalysed results | Summarization of the unalysed results |                                                                                              |
| Structured Output Parser4      | LangChain Output Parser         | Parses summarization output                         | Summarization of the unalysed results | Final report editor             |                                                                                              |
| OpenAI Chat Model5             | LangChain LLM Chat Model        | LLM backend for final report editing               | Final report editor             | Final report editor               |                                                                                              |
| Final report editor            | LangChain Agent                 | Produces final report draft                          | Structured Output Parser4       | Revisor and HTML formating agent  |                                                                                              |
| Structured Output Parser5      | LangChain Output Parser         | Parses final report draft                            | Final report editor             | Revisor and HTML formating agent  |                                                                                              |
| Revisor and HTML formating agent | LangChain Agent                 | Revises and formats report into HTML                 | Structured Output Parser5       | Completion agents (optional)       |                                                                                              |
| OpenAI Chat Model6             | LangChain LLM Chat Model        | LLM backend for revising and formatting             | Revisor and HTML formating agent | Revisor and HTML formating agent  |                                                                                              |
| Completion agents (optional)   | LangChain Agent                 | Optional final completions on report content         | Revisor and HTML formating agent | Edit Fields                     |                                                                                              |
| OpenAI Chat Model7             | LangChain LLM Chat Model        | LLM backend for completion agents                    | Completion agents (optional)    | Completion agents (optional)       |                                                                                              |
| Edit Fields                   | Set                            | Prepares email fields for sending                      | Completion agents (optional)    | Gmail                           |                                                                                              |
| Gmail                        | Gmail                          | Sends final HTML report via email                      | Edit Fields                   | None                            | Requires Gmail OAuth2 credentials configured in n8n.                                        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: To start the workflow manually.

2. **Add Google Sheets Node**  
   - Configure with Google Sheets credentials.  
   - Set Sheet ID and Tab to the source containing customer feedback data.

3. **Add Filter Node ("Get the first n rows")**  
   - Configure to filter the first 20 rows from Google Sheets output.

4. **Add Code Node ("Merging first sample rows")**  
   - Write JavaScript to aggregate filtered rows into a single dataset for prompt generation.

5. **Add LangChain Agent Node ("analysis topics proposal")**  
   - Connect input from the code node.  
   - Configure with instructions to generate 3–6 generalizable analysis prompts in JSON format.  
   - Use OpenAI Chat Model1 as the language model backend.

6. **Add LangChain LLM Chat Model Node ("OpenAI Chat Model1")**  
   - Configure with OpenAI or DeepSeek API credentials.  
   - Connect to the agent node.

7. **Add LangChain Output Parser Node ("Structured Output Parser1")**  
   - Configure to parse JSON prompt proposals from the agent.

8. **Add Merge Node ("Merge original table and the prompts")**  
   - Connect Google Sheets original data and parsed prompt proposals.  
   - Set merge mode to combine datasets for pairing.

9. **Add Function Node ("Prepare Prompts1")**  
   - Write JavaScript to inject all prompts into each feedback row, preparing for row-level analysis.

10. **Add LangChain Agent Node ("Unified AI Agent for analysis")**  
    - Connect input from the function node.  
    - Configure to analyze each row against all prompts.  
    - Use OpenAI Chat Model as backend.

11. **Add LangChain LLM Chat Model Node ("OpenAI Chat Model")**  
    - Configure with credentials.  
    - Connect to the agent node.

12. **Add LangChain Output Parser Node ("Structured Output Parser")**  
    - Configure to parse AI analysis output into structured fields.

13. **Add Code Node ("Transform results into columns")**  
    - Write JavaScript to convert AI output into columnar format.

14. **Add Code Node ("All unique elements merge")**  
    - Write JavaScript to aggregate unique analysis elements for prompt refinement.

15. **Add LangChain Agent Node ("proposal refinement agent")**  
    - Connect input from the previous code node.  
    - Configure to cluster synonyms and regenerate improved prompts.  
    - Use OpenAI Chat Model2 as backend.

16. **Add LangChain LLM Chat Model Node ("OpenAI Chat Model2")**  
    - Configure with credentials.

17. **Add LangChain Output Parser Node ("Structured Output Parser2")**  
    - Configure to parse refined prompt output.

18. **Add Merge Node ("Merge")**  
    - Combine refined prompts and original data for second iteration.

19. **Add LangChain Agent Node ("Second iteration of analysis")**  
    - Connect input from merge node.  
    - Configure to re-analyze rows with refined prompts.  
    - Use OpenAI Chat Model3 as backend.

20. **Add LangChain LLM Chat Model Node ("OpenAI Chat Model3")**  
    - Configure with credentials.

21. **Add LangChain Output Parser Node ("Structured Output Parser3")**  
    - Configure to parse second iteration output.

22. **Add Code Node ("wrap up the whole results into one Json file")**  
    - Write JavaScript to aggregate all refined analysis into a single JSON object.

23. **Add LangChain Agent Node ("Summarization of the unalysed results")**  
    - Connect input from previous code node.  
    - Configure to generate summaries and visualization data.  
    - Use OpenAI Chat Model4 as backend.

24. **Add LangChain LLM Chat Model Node ("OpenAI Chat Model4")**  
    - Configure with credentials.

25. **Add LangChain Output Parser Node ("Structured Output Parser4")**  
    - Configure to parse summarization output.

26. **Add LangChain Agent Node ("Final report editor")**  
    - Connect input from summarization parser.  
    - Configure to produce final report draft including charts and narratives.  
    - Use OpenAI Chat Model5 as backend.

27. **Add LangChain LLM Chat Model Node ("OpenAI Chat Model5")**  
    - Configure with credentials.

28. **Add LangChain Output Parser Node ("Structured Output Parser5")**  
    - Configure to parse final report draft.

29. **Add LangChain Agent Node ("Revisor and HTML formating agent")**  
    - Connect input from final report parser.  
    - Configure to revise and format report into HTML.  
    - Use OpenAI Chat Model6 as backend.

30. **Add LangChain LLM Chat Model Node ("OpenAI Chat Model6")**  
    - Configure with credentials.

31. **Add LangChain Agent Node ("Completion agents (optional)")**  
    - Optional node for final report enhancements.  
    - Use OpenAI Chat Model7 as backend.

32. **Add LangChain LLM Chat Model Node ("OpenAI Chat Model7")**  
    - Configure with credentials.

33. **Add Set Node ("Edit Fields")**  
    - Prepare email fields: recipient, subject, body (HTML report).

34. **Add Gmail Node**  
    - Configure with Gmail OAuth2 credentials.  
    - Set to send email with fields from Set node.

35. **Connect all nodes according to the logical flow described above.**

---

### 5. General Notes & Resources

| Note Content                                                                                      | Context or Link                                                                                      |
|-------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow automates semantic analysis of unstructured customer feedback using AI and charts.     | Workflow description and purpose.                                                                   |
| Requires DeepSeek API Key or OpenAI API Key and Gmail OAuth2 credentials.                        | Credential setup in n8n credential manager.                                                        |
| Google Sheet must contain at least 20 rows for prompt generation sample.                         | Data source precondition.                                                                           |
| QuickChart is used via URL encoding for visualization generation.                               | Visualization tool integrated in AI prompts and report.                                           |
| Final report is generated in HTML format and sent via Gmail.                                    | Email delivery node requires OAuth2 setup.                                                        |
| Customization options include data source, prompt definitions, LLM model swap, chart styling.   | See Customization Guide section in description.                                                    |
| Example results image referenced in original description (not included here).                    | Visual example of workflow output.                                                                 |
| For detailed LangChain node configuration, refer to n8n LangChain documentation.                | https://docs.n8n.io/integrations/builtin/nodes/n8n-nodes-langchain/                                |

---

This document provides a comprehensive reference for understanding, reproducing, and modifying the **Customer Feedback Analysis with AI, QuickChart & HTML Report Generator** workflow in n8n.