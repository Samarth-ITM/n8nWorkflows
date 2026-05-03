Create Research-Backed Articles with AI Planning, Linkup Search & GPT-5

https://n8nworkflows.xyz/workflows/create-research-backed-articles-with-ai-planning--linkup-search---gpt-5-8351


# Create Research-Backed Articles with AI Planning, Linkup Search & GPT-5

### 1. Workflow Overview

This workflow automates the creation of well-researched, high-quality articles by combining AI planning, web-based research, and advanced AI writing capabilities. It is designed for content creators, marketers, and teams seeking to produce fact-backed articles efficiently with integrated source referencing.

The workflow consists of the following logical blocks:

- **1.1 Input Reception:** Captures article title and writing guidelines through a user form.
- **1.2 AI Planning:** Uses a lightweight AI model to break down the article topic into structured research questions.
- **1.3 Research & Data Aggregation:** For each generated question, performs a web-based search via Linkup API to gather sourced insights, then aggregates all findings.
- **1.4 AI Writing:** Employs a powerful AI model to compose the final article based solely on the aggregated, sourced research and the initial instructions.
- **1.5 Output Preparation:** Formats and outputs the final article text.

Sticky notes throughout the workflow provide contextual guidance, credential setup reminders, and usage instructions.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

**Overview:**  
Collects user input for article title and specific writing guidelines through an n8n form trigger node.

**Nodes Involved:**  
- New article form  
- Prepare form values

**Node Details:**

- **New article form**  
  - Type: Form Trigger  
  - Role: Entry point, captures article title and guidelines with required fields.  
  - Configuration: Form title “New article” with two fields: “Article title” and “Article guidelines,” both required.  
  - Input: User submission via webhook.  
  - Output: JSON object with form fields.  
  - Edge cases: Missing required fields, webhook invocation failures.  
  - Sticky note: Highlights this form as the article request entry point.

- **Prepare form values**  
  - Type: Set  
  - Role: Standardizes form data by assigning “Title” and “Guidelines” variables from form input.  
  - Configuration: Assigns JSON keys from form response to named variables for downstream use.  
  - Input: Output of New article form.  
  - Output: JSON with `Title` and `Guidelines`.  
  - Edge cases: Empty or malformed inputs passed through.

---

#### 1.2 AI Planning

**Overview:**  
Uses a compact AI model to generate a set of 3–5 detailed, non-overlapping research questions that comprehensively cover the article topic, structured as a JSON array.

**Nodes Involved:**  
- GPT 5 mini  
- Structured Output Parser  
- Generate research questions  
- Split Out Questions

**Node Details:**

- **GPT 5 mini**  
  - Type: Langchain LLM Chat (OpenAI)  
  - Role: Generates research questions based on title and guidelines.  
  - Configuration: Model set to “gpt-5-mini” for efficient planning tasks.  
  - Input: Prompt with article title and guidelines.  
  - Output: Raw AI response with JSON array of questions.  
  - Credentials: OpenAI API (Duv’s OpenAI).  
  - Edge cases: Model timeout, malformed JSON output.  
  - Version: 1.2

- **Structured Output Parser**  
  - Type: Langchain Output Parser (Structured)  
  - Role: Parses the AI’s JSON response into structured data.  
  - Configuration: Uses a JSON schema example for validation and parsing.  
  - Input: Output from GPT 5 mini.  
  - Output: Parsed JSON array of question objects.  
  - Edge cases: Parsing errors if AI output deviates from schema.  
  - Version: 1.2

- **Generate research questions**  
  - Type: Langchain Chain LLM  
  - Role: Contains the prompt and logic to produce research questions in the correct format.  
  - Configuration: Prompt instructs the AI to produce 3–5 questions, starting broad then more specific.  
  - Input: Title and guidelines from Prepare form values.  
  - Output: Parsed JSON questions.  
  - Version: 1.5

- **Split Out Questions**  
  - Type: Split Out  
  - Role: Splits the array of questions into individual items for iterative processing.  
  - Input: Array of questions.  
  - Output: Single question per item.  
  - Edge cases: Empty or malformed arrays.

- Sticky note: Explains the block’s purpose as breaking down the topic into sub-questions.

---

#### 1.3 Research & Data Aggregation

**Overview:**  
For each research question, performs a web search using Linkup API to gather 5 specific, meaningful insights from reputable sources, then aggregates all results.

**Nodes Involved:**  
- Loop Over Questions  
- Query Linkup for insights  
- Format question and insights  
- Aggregate research content  
- Sticky Notes (credential reminder, block explanations)

**Node Details:**

- **Loop Over Questions**  
  - Type: Split In Batches  
  - Role: Iterates over individual questions to process them sequentially or in batches.  
  - Input: Single questions from Split Out Questions.  
  - Output: Each question passed to the research step.  
  - Edge cases: Large batch sizes causing timeouts.

- **Query Linkup for insights**  
  - Type: HTTP Request  
  - Role: Sends POST requests to Linkup’s search API to retrieve high-quality insights for each question.  
  - Configuration:  
    - URL: `https://api.linkup.so/v1/search`  
    - Method: POST  
    - Body Parameters: Query built to find 5 specific insights with concrete findings or examples, in a structured output format with schema validation.  
    - Authentication: HTTP Bearer Auth via stored Linkup API key credentials.  
  - Input: Current question text.  
  - Output: Structured JSON with insights, including insight text, source URL, and page title.  
  - Edge cases: API errors, authentication failures, rate limits, malformed responses.  
  - Version: 4.2

- **Format question and insights**  
  - Type: Set  
  - Role: Maps incoming data to named variables “Question” and “Insights” for clarity.  
  - Input: Response from Linkup.  
  - Output: JSON with explicit fields for downstream aggregation.

- **Aggregate research content**  
  - Type: Aggregate  
  - Role: Collects all individual research results into a single aggregated dataset under the field “Content to leverage.”  
  - Input: Multiple insights from each question’s research.  
  - Output: Single item containing all insights.

- Sticky notes:  
  - Remind to connect Linkup credentials properly.  
  - Explain the research purpose block.  
  - Caution regarding API key and generic credentials setup.

---

#### 1.4 AI Writing

**Overview:**  
Uses a large, high-quality AI model to compose the final article based solely on the aggregated research insights and initial user instructions, embedding hyperlinks to sources.

**Nodes Involved:**  
- Generate the AI output  
- GPT 5 chat  
- Article result  
- Sticky Note (writing block explanation)

**Node Details:**

- **Generate the AI output**  
  - Type: Langchain Chain LLM  
  - Role: Writes the full article integrating all sourced insights, adhering to the user’s guidelines.  
  - Configuration:  
    - Prompt includes the article title, guidelines, and JSON stringified aggregated insights.  
    - Output format instructions specify beginning with an H1 title and subheadings, embedding source links as hyperlinks.  
  - Input: Aggregated research content and initial form data.  
  - Output: Full article text.  
  - Version: 1.5

- **GPT 5 chat**  
  - Type: Langchain LLM Chat (OpenAI)  
  - Role: Executes the LLM writing task with the “gpt-5-chat-latest” model for best quality.  
  - Credentials: OpenAI API (Duv’s OpenAI).  
  - Input: Prompt from Generate the AI output.  
  - Output: Article text.  
  - Edge cases: Timeout, API errors, output truncation.

- **Article result**  
  - Type: Set  
  - Role: Sets the final article text in a named variable “Article” for output or further use.  
  - Input: Text from GPT 5 chat.  
  - Output: JSON containing the article.

- Sticky note: Describes this block as the final AI writing phase producing the sourced article.

---

#### 1.5 Output Preparation

**Overview:**  
Final node sets the article result for downstream consumption or output.

**Nodes Involved:**  
- Article result

**Node Details:**

- **Article result**  
  - Type: Set  
  - Role: Consolidates the final article text in a clearly named field.  
  - Configuration: Assigns the AI output text to `Article`.  
  - Input: Output from Generate the AI output.  
  - Output: JSON with full article content.  
  - Edge cases: Empty or malformed article text.

---

### 3. Summary Table

| Node Name                 | Node Type                                | Functional Role                         | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                      |
|---------------------------|----------------------------------------|---------------------------------------|------------------------------|------------------------------|------------------------------------------------------------------------------------------------|
| New article form           | Form Trigger                           | User input form for article request   |                              | Prepare form values           | ## Fill in this form to request a new article                                                  |
| Prepare form values        | Set                                   | Standardize form data fields           | New article form             | Generate research questions   |                                                                                                |
| GPT 5 mini                | Langchain LLM Chat (OpenAI)            | AI planner generating research questions | Prepare form values          | Generate research questions   |                                                                                                |
| Structured Output Parser   | Langchain Output Parser (Structured)   | Parses AI JSON output to structured questions | GPT 5 mini                  | Generate research questions   |                                                                                                |
| Generate research questions| Langchain Chain LLM                    | Contains prompt for AI question generation | Prepare form values, Structured Output Parser | Split Out Questions          | ## Breaking down the topic into sub-questions                                                  |
| Split Out Questions        | Split Out                             | Splits array of questions into items  | Generate research questions  | Loop Over Questions           |                                                                                                |
| Loop Over Questions        | Split In Batches                      | Iterates over questions for research  | Split Out Questions          | Query Linkup for insights, Aggregate research content | ## Retrieving insights from the web for each sub-question                                      |
| Query Linkup for insights  | HTTP Request                         | Queries Linkup API for insights per question | Loop Over Questions         | Format question and insights  | Connect your linkup.so credentials (adding your API key in the header or using stored generic credentials) |
| Format question and insights| Set                                  | Formats question and research output  | Query Linkup for insights    | Loop Over Questions           |                                                                                                |
| Aggregate research content | Aggregate                            | Aggregates all insights into one object | Loop Over Questions          | Generate the AI output        |                                                                                                |
| Generate the AI output     | Langchain Chain LLM                   | Composes final article integrating insights | Aggregate research content   | Article result                | ## AI step writing the final article based on the insights and initial request                |
| GPT 5 chat                | Langchain LLM Chat (OpenAI)            | Executes final article writing         | Generate the AI output       | Article result                |                                                                                                |
| Article result             | Set                                   | Sets final article text for output    | Generate the AI output       |                              |                                                                                                |
| Sticky Note                | Sticky Note                           | Reminder for Linkup credentials setup |                              |                              | Connect your linkup.so credentials (adding your API key in the header or using "generic credentials" that you've stored for Linkup). |
| Sticky Note1               | Sticky Note                           | Workflow explanation and usage notes  |                              |                              | # AI Article Research & Writing Team... (full content in block 1.1)                            |
| Sticky Note2               | Sticky Note                           | Explanation of web insight retrieval  |                              |                              | ## Retrieving insights from the web for each sub-question                                      |
| Sticky Note3               | Sticky Note                           | Explanation of AI writing step         |                              |                              | ## AI step writing the final article based on the insights and initial request                 |
| Sticky Note4               | Sticky Note                           | Explanation of question breakdown      |                              |                              | ## Breaking down the topic into sub-questions                                                  |
| Sticky Note5               | Sticky Note                           | Explanation of form usage               |                              |                              | ## Fill in this form to request a new article                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node “New article form”:**  
   - Set form title: “New article”  
   - Add two required fields:  
     - “Article title” (text input)  
     - “Article guidelines” (text input)  
   - This node acts as the webhook entry point.

2. **Add a Set node “Prepare form values”:**  
   - Assign variables:  
     - `Title` = `{{$json["Article title"]}}`  
     - `Guidelines` = `{{$json["Article guidelines"]}}`  
   - Connect input from “New article form”.

3. **Add a Langchain Chain LLM node “Generate research questions”:**  
   - Use OpenAI credentials.  
   - Model: “gpt-5-mini” or equivalent small planning model.  
   - Prompt: Provide the article title and guidelines, instruct AI to output 3–5 non-overlapping research questions in a JSON array format.  
   - Enable structured output parser.

4. **Add Langchain Output Parser node “Structured Output Parser”:**  
   - Provide example JSON schema matching expected questions array.  
   - Connect input from “Generate research questions” LLM output.

5. **Add a Split Out node “Split Out Questions”:**  
   - Field to split out: `output` from parsed questions array.  
   - Connect input from “Generate research questions”.

6. **Add a Split In Batches node “Loop Over Questions”:**  
   - Purpose: process each question individually.  
   - Connect input from “Split Out Questions”.

7. **Add an HTTP Request node “Query Linkup for insights”:**  
   - Method: POST  
   - URL: `https://api.linkup.so/v1/search`  
   - Authentication: HTTP Bearer Auth using Linkup API Key credential.  
   - Body parameters:  
     - `q`: Formatted prompt asking for 5 specific insights about `{{ $json.question }}`  
     - `depth`: “standard”  
     - `outputType`: “structured”  
     - `structuredOutputSchema`: JSON schema describing insights array with fields: insight, url, title  
     - `fromDate`: Optional date filter (can be dynamic)  
     - `includeImages`: false  
   - Connect input from “Loop Over Questions”.

8. **Add a Set node “Format question and insights”:**  
   - Assign:  
     - `Question` = `{{$json["question"]}}`  
     - `Insights` = `{{$json["insights"]}}` (from Linkup response)  
   - Connect input from “Query Linkup for insights”.

9. **Feed output of “Format question and insights” back into “Loop Over Questions” for batch iteration.**

10. **Add an Aggregate node “Aggregate research content”:**  
    - Aggregate all items into one under field “Content to leverage”.  
    - Connect input from “Loop Over Questions”.

11. **Add Langchain Chain LLM node “Generate the AI output”:**  
    - Use OpenAI credentials.  
    - Model: “gpt-5-chat-latest” or equivalent large chat model.  
    - Prompt includes:  
      - Article title and guidelines from “Prepare form values”.  
      - JSON stringified aggregated insights from “Aggregate research content”.  
      - Instructions to write a full article, embedding source hyperlinks, starting with an H1 and subheadings.  
    - Connect input from “Aggregate research content”.

12. **Add Langchain LLM Chat node “GPT 5 chat”:**  
    - Model: “gpt-5-chat-latest”.  
    - Credentials: OpenAI API.  
    - Connect input from “Generate the AI output”.

13. **Add a Set node “Article result”:**  
    - Assign `Article` = `{{$json.text}}` (final article text).  
    - Connect input from “GPT 5 chat”.

14. **Add Sticky Note nodes as needed for documentation and reminders:**
    - Reminder to connect Linkup API credentials properly.  
    - Explanation of each logical block and usage tips.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                    | Context or Link                                                                                       |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| This workflow mimics a professional writing process to create high-quality, sourced articles. It plans questions, researches with Linkup, then drafts the article based on reliable sources.                                                                                                                                                                                                                      | Workflow introduction and explanation (Sticky Note1)                                                |
| Connect your Linkup API key either by adding it in the HTTP header or using “generic credentials” saved in n8n for authentication.                                                                                                                                                                                                                                                                               | Credential setup reminder (Sticky Note near Query Linkup node)                                      |
| Use a smaller AI model for planning research questions to save cost and time, and a larger model for final article writing for quality.                                                                                                                                                                                                                                                                          | Best practice recommendation for model usage                                                       |
| The prompt for research instructs Linkup to find concrete findings, statistics, or examples—not generic background—to ensure factual strength.                                                                                                                                                                                                                                                                     | Prompt design note for research queries                                                             |
| The final AI writing output is instructed to embed source URLs as hyperlinks smoothly to maintain transparency and credibility.                                                                                                                                                                                                                                                                                   | Writing format guideline                                                                            |
| This template was created by Guillaume Duvernay and is suitable for teams and individuals focused on data-backed content creation.                                                                                                                                                                                                                                                                               | Project credits                                                                                     |

---

**Disclaimer:** The provided text originates exclusively from an n8n automated workflow. All data processed is legal and publicly available. The workflow strictly complies with content policies and does not contain illegal, offensive, or protected materials.