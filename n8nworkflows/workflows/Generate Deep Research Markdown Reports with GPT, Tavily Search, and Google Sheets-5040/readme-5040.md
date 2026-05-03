Generate Deep Research Markdown Reports with GPT, Tavily Search, and Google Sheets

https://n8nworkflows.xyz/workflows/generate-deep-research-markdown-reports-with-gpt--tavily-search--and-google-sheets-5040


# Generate Deep Research Markdown Reports with GPT, Tavily Search, and Google Sheets

### 1. Workflow Overview

This workflow automates the generation of deep research Markdown reports by integrating GPT-based AI agents (via OpenRouter language models), Tavily search API, and Google Sheets for data storage and retrieval. It is designed for content creators, researchers, and marketers who want to produce structured blog or research articles efficiently by leveraging AI-driven topic planning, content generation, and source aggregation.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Captures user input from a form submission that triggers the entire report generation.
- **1.2 Topic Planning:** Uses AI to generate a list of relevant blog/research topics based on the input.
- **1.3 Introduction Drafting:** AI crafts an introductory section based on planned topics.
- **1.4 Topic Splitting and Processing:** Splits topics for parallel processing and content generation.
- **1.5 Source Gathering and Management:** Retrieves and stores source data from Google Sheets and Tavily search API.
- **1.6 Content Writing:** AI generates detailed content sections for each topic.
- **1.7 Table of Contents Creation:** AI summarizes the structure into a table of contents.
- **1.8 Aggregation and Output:** Aggregates generated content and stores results back into Google Sheets.
- **1.9 Looping and Batch Processing:** Handles multiple items/topics iteratively to scale content creation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** Waits for external form submissions to trigger the workflow process.
- **Nodes Involved:** On form submission
- **Node Details:**
  - Type: Form Trigger
  - Role: Entry point for the workflow; listens for form webhook submissions.
  - Configuration: Uses a specific webhook ID to receive form data.
  - Input Connections: None (trigger node)
  - Output Connections: Connects to "Plan Topics" node
  - Edge Cases: Webhook not triggered, missing or malformed form data
  - Version: 2.2

#### 1.2 Topic Planning

- **Overview:** Generates a set of blog or research topics using an AI agent.
- **Nodes Involved:** Plan Topics, OpenRouter Chat Model, 5 Topics, Split Out, Set Topics, Split Out1, Merge1, Merge
- **Node Details:**

  - **Plan Topics**
    - Type: LangChain Agent
    - Role: Orchestrates AI agents for topic generation.
    - Configuration: Uses AI models connected via "OpenRouter Chat Model".
    - Inputs: Triggered by "On form submission"
    - Outputs: Intro, Split Out, Set Topics nodes
    - Edge Cases: AI model failure, rate limits, malformed output
    - Version: 1.8

  - **OpenRouter Chat Model**
    - Type: LangChain OpenRouter Chat Model
    - Role: Provides the language model API interface for AI generation.
    - Configuration: Default model settings, linked with "Plan Topics".
    - Input: From "Plan Topics" node's ai_languageModel port
    - Output: To "Plan Topics"
    - Edge Cases: Authentication errors, API timeouts

  - **5 Topics**
    - Type: LangChain Output Parser Structured
    - Role: Parses AI output into a structured list of 5 topics.
    - Configuration: Preset to parse topics from AI text.
    - Input: AI output from "OpenRouter Chat Model"
    - Output: Back to "Plan Topics" (ai_outputParser port)
    - Edge Cases: Parsing errors due to unexpected AI output format

  - **Split Out / Split Out1**
    - Type: Split Out
    - Role: Splits data arrays to individual items for parallel processing.
    - Input: From "Plan Topics" and "Set Topics"
    - Output: To "Merge1"
    - Edge Cases: Empty input arrays

  - **Set Topics**
    - Type: Set
    - Role: Prepares or standardizes topic data for processing.
    - Input: From "Plan Topics"
    - Output: To "Split Out1"
    - Edge Cases: Missing or invalid data

  - **Merge1 / Merge**
    - Type: Merge
    - Role: Merges split items back into arrays for downstream processing.
    - Input: From split nodes and other merges
    - Output: To "Merge" (which leads to "Loop Over Items")
    - Edge Cases: Mismatched data arrays

#### 1.3 Introduction Drafting

- **Overview:** AI generates an introductory text for the planned topics.
- **Nodes Involved:** Intro, OpenRouter Chat Model1, Title, Intro, Chapters, Send Intro
- **Node Details:**

  - **Intro**
    - Type: LangChain Agent
    - Role: Generates introduction content for the blog/report.
    - Input: From "Plan Topics"
    - Output: "Send Intro" and "OpenRouter Chat Model1"
    - Edge Cases: AI generation failure

  - **OpenRouter Chat Model1**
    - Type: LangChain OpenRouter Chat Model
    - Role: Provides AI language model access for intro generation.
    - Input: From "Intro" (ai_languageModel port)
    - Output: To "Intro"
    - Edge Cases: API errors or rate limiting

  - **Title, Intro, Chapters**
    - Type: LangChain Output Parser Structured
    - Role: Parses the AI-generated intro and chapter titles.
    - Input: From "OpenRouter Chat Model1"
    - Output: To "Intro" (ai_outputParser port)
    - Edge Cases: Parsing failures

  - **Send Intro**
    - Type: Google Sheets
    - Role: Saves the introduction content to Google Sheets.
    - Input: From "Intro"
    - Output: To "Merge"
    - Edge Cases: Google Sheets API authentication or quota issues

#### 1.4 Topic Splitting and Processing

- **Overview:** Splits the planned topics into individual units for detailed processing and content generation.
- **Nodes Involved:** Split Out, Merge1, Merge, Loop Over Items
- **Node Details:**

  - **Split Out**
    - Splits the topics array into individual items for batch processing.
  - **Merge1 / Merge**
    - Recombines data streams as needed.
  - **Loop Over Items**
    - Type: Split In Batches
    - Role: Iterates over batches of topics for scalable processing.
    - Edge Cases: Empty batches, batch size misconfigurations

#### 1.5 Source Gathering and Management

- **Overview:** Retrieves research sources from Google Sheets and the Tavily search API; manages source data for content enrichment.
- **Nodes Involved:** Get Sources, Sources, Send Sources, Tavily5, Split Out7, Code5, Aggregate12, Google Sheets, Get All Content
- **Node Details:**

  - **Get Sources**
    - Type: Google Sheets
    - Role: Reads source URLs and data from a configured Google Sheets document.
    - Edge Cases: Sheet not found, read permissions

  - **Sources**
    - Type: LangChain Agent
    - Role: Processes and enriches source data.
    - Input: From "Get Sources"
    - Output: To "Send Sources"

  - **Send Sources**
    - Type: Google Sheets
    - Role: Writes processed sources back to Google Sheets.
    - Input: From "Sources"
    - Output: To "Get All Content"

  - **Tavily5**
    - Type: HTTP Request
    - Role: Queries Tavily API for web search related to topics.
    - Input: From "Loop Over Items"
    - Edge Cases: Network errors, API key expiration

  - **Split Out7**
    - Splits Tavily API responses for processing.

  - **Code5**
    - Type: Code
    - Role: Transforms or processes Tavily API results.
    - Input: From "Split Out7"
    - Output: To "Writer5"

  - **Aggregate12**
    - Aggregates processed data before saving.

  - **Google Sheets**
    - Writes aggregated data back to sheets.

  - **Get All Content**
    - Reads all collected content for final synthesis.

#### 1.6 Content Writing

- **Overview:** Generates detailed report content for each topic using AI.
- **Nodes Involved:** Writer5, OpenRouter Chat Model8, Aggregate10, Aggregate11, Merge8, Get Sections
- **Node Details:**

  - **Writer5**
    - Type: LangChain Agent
    - Role: Writes detailed sections based on topic and source data.
    - Input: From "Code5" processed Tavily data
    - Output: To "Aggregate10"

  - **OpenRouter Chat Model8**
    - AI model node linked to "Writer5"

  - **Aggregate10 / Aggregate11**
    - Aggregate partial outputs from "Writer5" for merging.

  - **Merge8**
    - Merges aggregated content streams.

  - **Get Sections**
    - Type: Code
    - Role: Extracts and formats sections from merged content.
    - Output: Feeds back to "Merge8" for completeness.

#### 1.7 Table of Contents Creation

- **Overview:** Generates a structured table of contents from the aggregated content.
- **Nodes Involved:** Table of Contents, OpenRouter Chat Model7, Send ToC
- **Node Details:**

  - **Table of Contents**
    - Type: LangChain Agent
    - Role: Creates a Markdown table of contents.
    - Input: From "Get All Content"
    - Output: To "Send ToC"

  - **OpenRouter Chat Model7**
    - Connects to the Table of Contents agent for generation

  - **Send ToC**
    - Type: Google Sheets
    - Role: Saves the table of contents to Google Sheets.

#### 1.8 Aggregation and Output

- **Overview:** Collects and stores all generated outputs including intro, sources, content, and table of contents into Google Sheets.
- **Nodes Involved:** Send Intro, Send Sources, Send ToC, Google Sheets
- **Node Details:**

  - Nodes save respective data parts to Google Sheets with error handling for API issues.

#### 1.9 Looping and Batch Processing

- **Overview:** Manages batch processing of topics and content to handle multiple inputs efficiently.
- **Nodes Involved:** Loop Over Items, Aggregate12, Tavily5, Split Out7, Code5, Writer5, Merge8
- **Node Details:**

  - Loop Over Items splits data into manageable batches.
  - Aggregates and merges results for final output.

---

### 3. Summary Table

| Node Name            | Node Type                                  | Functional Role                       | Input Node(s)           | Output Node(s)           | Sticky Note                         |
|----------------------|--------------------------------------------|------------------------------------|-------------------------|--------------------------|-----------------------------------|
| On form submission   | Form Trigger                              | Entry point, capture form data      | None                    | Plan Topics              |                                   |
| Plan Topics          | LangChain Agent                           | AI topic generation                 | On form submission      | Intro, Split Out, Set Topics |                                   |
| OpenRouter Chat Model | LangChain OpenRouter Chat Model           | AI language model for topic planning| Plan Topics             | Plan Topics              |                                   |
| 5 Topics             | LangChain Output Parser Structured         | Parse AI topics output              | OpenRouter Chat Model   | Plan Topics              |                                   |
| Split Out            | Split Out                                 | Split topic arrays                  | Plan Topics             | Merge1                   |                                   |
| Set Topics           | Set                                       | Prepare topics data                 | Plan Topics             | Split Out1               |                                   |
| Split Out1           | Split Out                                 | Split processed topics             | Set Topics              | Merge1                   |                                   |
| Merge1               | Merge                                     | Merge split topics                 | Split Out, Split Out1   | Merge                    |                                   |
| Merge                | Merge                                     | Aggregate topics for looping       | Merge1                  | Loop Over Items           |                                   |
| Intro                | LangChain Agent                           | Generate introduction text          | Plan Topics             | Send Intro, OpenRouter Chat Model1 |                                   |
| OpenRouter Chat Model1| LangChain OpenRouter Chat Model           | AI model for intro generation       | Intro                   | Intro                    |                                   |
| Title, Intro, Chapters| LangChain Output Parser Structured         | Parse intro and chapter titles      | OpenRouter Chat Model1  | Intro                    |                                   |
| Send Intro           | Google Sheets                             | Store introduction                  | Intro                   | Merge                    |                                   |
| Loop Over Items      | Split In Batches                          | Batch processing of topics          | Merge                   | Tavily5, Aggregate12      |                                   |
| Tavily5              | HTTP Request                             | Query Tavily API for research       | Loop Over Items         | Split Out7               |                                   |
| Split Out7           | Split Out                                 | Split Tavily API results            | Tavily5                 | Code5                    |                                   |
| Code5                | Code                                      | Process Tavily data                 | Split Out7              | Writer5                  |                                   |
| Writer5              | LangChain Agent                           | Write detailed content              | Code5                   | Aggregate10              |                                   |
| OpenRouter Chat Model8| LangChain OpenRouter Chat Model           | AI model for content writing        | Writer5                  | Writer5                  |                                   |
| Aggregate10          | Aggregate                                 | Aggregate content pieces            | Writer5                  | Get Sections, Merge8     |                                   |
| Aggregate11          | Aggregate                                 | Aggregate more content pieces       | Code5                    | Merge8                   |                                   |
| Merge8               | Merge                                     | Merge all content pieces            | Aggregate10, Aggregate11, Get Sections | Loop Over Items      |                                   |
| Get Sections         | Code                                      | Extract and format content sections | Aggregate10              | Merge8                   |                                   |
| Aggregate12          | Aggregate                                 | Collect processed data              | Loop Over Items          | Google Sheets            |                                   |
| Google Sheets        | Google Sheets                             | Store aggregated data               | Aggregate12              | Get Sources              |                                   |
| Get Sources          | Google Sheets                             | Read source data                   | Google Sheets            | Sources                  |                                   |
| Sources              | LangChain Agent                           | Process and enrich source data      | Get Sources              | Send Sources             |                                   |
| Send Sources         | Google Sheets                             | Save sources back                   | Sources                  | Get All Content          |                                   |
| Get All Content      | Google Sheets                             | Read all content for synthesis      | Send Sources             | Table of Contents        |                                   |
| Table of Contents    | LangChain Agent                           | Generate table of contents          | Get All Content          | Send ToC                 |                                   |
| OpenRouter Chat Model7| LangChain OpenRouter Chat Model           | AI model for ToC generation         | Table of Contents        | Table of Contents        |                                   |
| Send ToC             | Google Sheets                             | Save table of contents              | Table of Contents        |                          |                                   |
| Sticky Note          | Sticky Note                              | (Empty/No comment)                  |                         |                          |                                   |
| Sticky Note1         | Sticky Note                              | (Empty/No comment)                  |                         |                          |                                   |
| Sticky Note2         | Sticky Note                              | (Empty/No comment)                  |                         |                          |                                   |
| Sticky Note6         | Sticky Note                              | (Empty/No comment)                  |                         |                          |                                   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger Node**
   - Type: Form Trigger
   - Configuration: Set webhook ID to listen for form submissions.
   - Purpose: Entry point for user input.

2. **Add "Plan Topics" LangChain Agent Node**
   - Connect input from Form Trigger.
   - Configure to use OpenRouter Chat Model for AI topic planning.
   - Outputs connected to "Intro", "Split Out", and "Set Topics".

3. **Add "OpenRouter Chat Model" Node for Plan Topics**
   - Configure OpenRouter API credentials.
   - Connect to "Plan Topics" node (ai_languageModel port).

4. **Add "5 Topics" Output Parser Node**
   - Connect from "OpenRouter Chat Model" (ai_outputParser port).
   - Parses topics into structured format.
   - Send output back to "Plan Topics".

5. **Add "Split Out" Node**
   - Connect from "Plan Topics".
   - Splits topics array into separate items.
   - Output connects to "Merge1".

6. **Add "Set Topics" Node**
   - Connect from "Plan Topics".
   - Prepares and standardizes topic data.
   - Output connects to "Split Out1".

7. **Add "Split Out1" Node**
   - Connect from "Set Topics".
   - Splits standardized topic data.
   - Output connects to "Merge1".

8. **Add "Merge1" Node**
   - Connect inputs from "Split Out" and "Split Out1".
   - Output connects to "Merge".

9. **Add "Merge" Node**
   - Connect from "Merge1".
   - Output connects to "Loop Over Items".

10. **Add "Intro" LangChain Agent Node**
    - Connect input from "Plan Topics".
    - Configure to use "OpenRouter Chat Model1" for intro generation.
    - Outputs connect to "Send Intro" and back to "OpenRouter Chat Model1".

11. **Add "OpenRouter Chat Model1" Node**
    - Configure OpenRouter API credentials.
    - Connect to "Intro" (ai_languageModel port).

12. **Add "Title, Intro, Chapters" Output Parser**
    - Connect output from "OpenRouter Chat Model1".
    - Parses intro and chapters.
    - Output connects back to "Intro" (ai_outputParser port).

13. **Add "Send Intro" Google Sheets Node**
    - Connect from "Intro".
    - Configure Google Sheets credentials and target spreadsheet.
    - Output connects to "Merge".

14. **Add "Loop Over Items" Node**
    - Type: Split In Batches
    - Connect from "Merge".
    - Outputs connect to "Tavily5" and "Aggregate12".

15. **Add "Tavily5" HTTP Request Node**
    - Connect from "Loop Over Items".
    - Configure to query Tavily Search API.
    - Output connects to "Split Out7".

16. **Add "Split Out7" Node**
    - Connect from "Tavily5".
    - Splits API results.
    - Output connects to "Code5".

17. **Add "Code5" Node**
    - Connect from "Split Out7".
    - Write custom JS to process Tavily results.
    - Output connects to "Writer5".

18. **Add "Writer5" LangChain Agent Node**
    - Connect from "Code5".
    - Configure with "OpenRouter Chat Model8" for detailed content writing.
    - Output connects to "Aggregate10".

19. **Add "OpenRouter Chat Model8" Node**
    - Configure OpenRouter API credentials.
    - Connect to "Writer5" (ai_languageModel port).

20. **Add "Aggregate10" Node**
    - Connect from "Writer5".
    - Output connects to "Get Sections" and "Merge8".

21. **Add "Aggregate11" Node**
    - Connect from "Code5".
    - Output connects to "Merge8".

22. **Add "Merge8" Node**
    - Connect inputs from "Aggregate10", "Aggregate11", and "Get Sections".
    - Output connects to "Loop Over Items".

23. **Add "Get Sections" Code Node**
    - Connect from "Aggregate10".
    - Write JS code to extract and format content sections.
    - Output connects back to "Merge8".

24. **Add "Aggregate12" Node**
    - Connect from "Loop Over Items".
    - Output connects to "Google Sheets" node.

25. **Add "Google Sheets" Node**
    - Connect from "Aggregate12".
    - Configure to save aggregated data.
    - Output connects to "Get Sources".

26. **Add "Get Sources" Google Sheets Node**
    - Connect from "Google Sheets".
    - Configure to read source data.
    - Output connects to "Sources".

27. **Add "Sources" LangChain Agent Node**
    - Connect from "Get Sources".
    - Configure with "OpenRouter Chat Model7".
    - Output connects to "Send Sources".

28. **Add "OpenRouter Chat Model7" Node**
    - Configure OpenRouter API credentials.
    - Connect to both "Sources" and "Table of Contents".

29. **Add "Send Sources" Google Sheets Node**
    - Connect from "Sources".
    - Configure to save processed sources.
    - Output connects to "Get All Content".

30. **Add "Get All Content" Google Sheets Node**
    - Connect from "Send Sources".
    - Reads all content for final summary.
    - Output connects to "Table of Contents".

31. **Add "Table of Contents" LangChain Agent Node**
    - Connect from "Get All Content".
    - Uses "OpenRouter Chat Model7" to generate ToC.
    - Output connects to "Send ToC".

32. **Add "Send ToC" Google Sheets Node**
    - Connect from "Table of Contents".
    - Saves ToC to spreadsheet.

33. **Add sticky notes as needed for documentation.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                      | Context or Link                                                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------|
| Workflow integrates GPT via OpenRouter API, Tavily Search API, and Google Sheets for content automation.                                                       | Core workflow integration points.                                                                  |
| Form trigger starts the process, allowing external user input to dynamically generate reports.                                                                  | Entry point for user interaction.                                                                  |
| Use valid credentials for OpenRouter and Google Sheets nodes to avoid authentication errors.                                                                     | Credential setup required before workflow activation.                                              |
| Tavily5 node requires API key and proper network connectivity to fetch search results.                                                                            | Ensure Tavily API access is configured correctly.                                                  |
| The workflow uses batch processing to handle multiple topics efficiently; adjust batch size in "Loop Over Items" node for performance tuning.                   | Performance tuning tip.                                                                             |
| Output parsers rely on consistent AI output format; unexpected changes in AI responses can cause parsing failures.                                               | Monitor AI output for structural consistency.                                                     |
| Google Sheets nodes require appropriate sheet IDs and range configuration.                                                                                       | Sheet configuration is critical for data storage and retrieval.                                   |
| Workflow designed for Markdown report generation with structured sections, intros, and tables of contents.                                                      | Target output format and use case.                                                                 |

---

**Disclaimer:** The provided description and analysis are exclusively derived from a workflow automated with n8n, respecting all current content policies. No illegal, offensive, or protected content is included. All processed data is legal and public.