Create Fact-Based Blog Outlines with GPT-4o & Tavily Search Data

https://n8nworkflows.xyz/workflows/create-fact-based-blog-outlines-with-gpt-4o---tavily-search-data-9058


# Create Fact-Based Blog Outlines with GPT-4o & Tavily Search Data

### 1. Workflow Overview

This workflow automates the generation of a fact-based blog outline using a combination of AI (OpenAI’s GPT-4o) and up-to-date web research via the Tavily API. It is designed to take a user-provided keyword, decompose it into structured subtopics with research questions, fetch real-time answers from the web, and compile the enriched information into a Markdown outline suitable for content creation.

**Target use cases:**
- Content marketers and SEO specialists who want to create data-driven blog outlines.
- Writers needing accurate, research-backed topic structures.
- Automation enthusiasts integrating AI and live web data for content workflows.

**Logical Blocks:**

- **1.1 Input Reception:** Captures the keyword from a user form.
- **1.2 Research Question Generation:** Uses GPT-4o to generate subtopics and related research questions.
- **1.3 Question Splitting:** Splits the list of questions into individual items for parallel processing.
- **1.4 Research Answering:** Queries Tavily API to fetch answers for each research question.
- **1.5 Answer Aggregation:** Combines answers with their respective sections and questions.
- **1.6 Outline Aggregation:** Aggregates all enriched sections back into a single JSON array.
- **1.7 Markdown Conversion:** Transforms the JSON outline into a Markdown formatted document.
- **1.8 Workflow Continuation:** Placeholder node to extend the workflow (e.g., export or further processing).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
Receives the main keyword from the user through a web form to initiate the outline generation process.

**Nodes Involved:**  
- Enter keyword

**Node Details:**

- **Enter keyword**  
  - Type: Form Trigger  
  - Role: Captures user input via a form with a single required field labeled "Keyword".  
  - Configuration: Form titled "Outline generator", with a description explaining the workflow’s purpose.  
  - Input: User-submitted keyword string.  
  - Output: JSON containing the keyword under `$json.Keyword`.  
  - Failure modes: Form submission errors, webhook connectivity issues.  
  - Notes: Webhook ID is set to receive external HTTP POST requests.

#### 2.2 Research Question Generation

**Overview:**  
Generates 5-6 subtopics and corresponding research questions related to the keyword using GPT-4o.

**Nodes Involved:**  
- Generate research questions  
- Sticky Note (explanatory)

**Node Details:**

- **Generate research questions**  
  - Type: OpenAI (Langchain) node  
  - Role: Acts as a research assistant, taking the keyword and outputting a JSON object containing an outline of sections with associated research questions.  
  - Configuration: Model set to "gpt-4o", system message instructs strict JSON output with 5-6 sections/questions.  
  - Key expressions: Uses `{{ $json.Keyword }}` to dynamically insert the entered keyword.  
  - Input: JSON from the form trigger containing the keyword.  
  - Output: Parsed JSON array under `message.content.outline`.  
  - Failure modes: API authentication errors, rate limits, formatting/parsing errors if output deviates from JSON.  
  - Credentials: OpenAI API key required.  

- **Sticky Note2** (contextual)  
  - Explains the node’s purpose: splitting the keyword into subtopics and generating research questions.

#### 2.3 Question Splitting

**Overview:**  
Splits the JSON array of research questions into separate items to process each question individually.

**Nodes Involved:**  
- Split out list of questions into separate items  
- Sticky Note3

**Node Details:**

- **Split out list of questions into separate items**  
  - Type: Split Out  
  - Role: Takes the outline array and creates one workflow item per research question.  
  - Configuration: Splits on the field `message.content.outline`.  
  - Input: JSON outline array from the OpenAI node.  
  - Output: Multiple items, each representing a single section/question pair.  
  - Failure modes: Missing or malformed JSON array.  
  - Notes: Enables parallel or sequential processing of questions.

#### 2.4 Research Answering

**Overview:**  
For each research question, queries the Tavily API to fetch relevant, up-to-date answers derived from real web searches.

**Nodes Involved:**  
- Answer research questions  
- Sticky Note4

**Node Details:**

- **Answer research questions**  
  - Type: Tavily community node  
  - Role: Sends the individual research question to Tavily’s API with advanced search depth and requests an answer summary.  
  - Configuration:  
    - Query parameter dynamically set to `{{ $json.question }}`.  
    - Options include topic "general", "advanced" search depth, and answer inclusion.  
  - Input: Single research question item from splitter node.  
  - Output: JSON containing answer text under `$json.answer`.  
  - Failure modes: API key issues, network timeouts, empty or irrelevant results.  
  - Credentials: Tavily API key required.  
  - Notes: Can be replaced by HTTP node calling Tavily API directly.

#### 2.5 Answer Aggregation

**Overview:**  
Combines the section title, research question, and its answer into a structured JSON object for each item.

**Nodes Involved:**  
- Add answers to sections  
- Sticky Note5

**Node Details:**

- **Add answers to sections**  
  - Type: Set  
  - Role: Assigns new JSON fields combining section title, question, and retrieved answer.  
  - Configuration:  
    - `section_title` set from the original section headline.  
    - `section_question` set from the question text.  
    - `answer` set from Tavily’s answer.  
  - Input: Merges data from the split questions node and the Tavily answer node via the workflow context.  
  - Output: Structured JSON object per section with enriched data.  
  - Failure modes: Missing input fields, expression evaluation errors.

#### 2.6 Outline Aggregation

**Overview:**  
Aggregates the individually enriched section objects back into a single list for further processing.

**Nodes Involved:**  
- Turn into one big item  
- Sticky Note6

**Node Details:**

- **Turn into one big item**  
  - Type: Aggregate  
  - Role: Collects all individual items and merges their JSON data into one array under the field `outline`.  
  - Configuration: Aggregation mode set to gather all item data.  
  - Input: Multiple JSON items from the previous set node.  
  - Output: Single JSON object with an array of all sections and answers.  
  - Failure modes: Empty input, aggregation errors.

#### 2.7 Markdown Conversion

**Overview:**  
Converts the JSON outline into a Markdown formatted string suitable for content generation or export.

**Nodes Involved:**  
- Convert JSON to markdown  
- Sticky Note7

**Node Details:**

- **Convert JSON to markdown**  
  - Type: Code (JavaScript)  
  - Role: Transforms each section into a Markdown H2 heading, bolds the question, and places the answer below.  
  - Configuration: Custom JavaScript code processes the `outline` array.  
  - Input: Aggregated JSON outline array.  
  - Output: Single Markdown string under `$json.markdown`.  
  - Failure modes: Missing fields in JSON, runtime errors in JS code.  
  - Notes: Questions are included only as bold text for reference; main focus is on answers.

#### 2.8 Workflow Continuation

**Overview:**  
A placeholder node to continue the workflow, typically for exporting the Markdown outline or further AI processing.

**Nodes Involved:**  
- Continue workflow  
- Sticky Note8

**Node Details:**

- **Continue workflow**  
  - Type: NoOp (no operation)  
  - Role: Marks the endpoint for this workflow or a handoff point for subsequent nodes (e.g., article generation, Google Docs export).  
  - Configuration: None.  
  - Input: Markdown output from previous node.  
  - Output: Passes data forward unchanged.  
  - Failure modes: None.

---

### 3. Summary Table

| Node Name                               | Node Type                       | Functional Role                                  | Input Node(s)                          | Output Node(s)                          | Sticky Note                                                                                                                      |
|----------------------------------------|--------------------------------|-------------------------------------------------|--------------------------------------|----------------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| Enter keyword                          | Form Trigger                   | Receives keyword input via web form             | —                                    | Generate research questions             | Enter your main topic into the form. The AI will split it up into 5-6 different subtopics                                        |
| Generate research questions            | OpenAI (Langchain)             | Generates subtopics and research questions      | Enter keyword                       | Split out list of questions into separate items | Create subtopics and generate research questions. This AI node takes the keyword and turns it into 5-6 subtopics.                |
| Split out list of questions into separate items | Split Out                      | Splits questions array into separate workflow items | Generate research questions          | Answer research questions              | Split out questions. In this node we split all the research questions into separate items so we can research them             |
| Answer research questions              | Tavily community node          | Fetches answers from Tavily API for each question | Split out list of questions into separate items | Add answers to sections               | Get answers to research questions using Tavily API. This node searches up-to-date results and synthesizes answers.             |
| Add answers to sections                | Set                           | Combines section titles, questions, and answers | Answer research questions            | Turn into one big item                 | Add answers to JSON. Now we have a section title, main question, and answer.                                                    |
| Turn into one big item                 | Aggregate                     | Aggregates all enriched section items into one list | Add answers to sections              | Convert JSON to markdown               | Turn into one list. Combine all different sections into one big list again.                                                     |
| Convert JSON to markdown               | Code (JavaScript)             | Converts JSON outline to Markdown format         | Turn into one big item               | Continue workflow                     | Convert JSON into markdown. Converts section titles into headings and adds researched answers.                                  |
| Continue workflow                     | NoOp                          | Placeholder for further workflow steps           | Convert JSON to markdown             | —                                      | Continue with the workflow. Add another AI node or export options to use the enriched outline.                                  |
| Sticky Note                           | Sticky Note                   | Overview and detailed explanation of the workflow | —                                    | —                                      | Turn your keyword research into a clear, fact-based content outline with this workflow. See detailed instructions and links.   |
| Sticky Note1                          | Sticky Note                   | Explains keyword input block                      | —                                    | —                                      | Enter your main topic into the form. The AI will split it up into 5-6 different subtopics                                        |
| Sticky Note2                          | Sticky Note                   | Explains research question generation block      | —                                    | —                                      | Create subtopics and generate research questions.                                                                             |
| Sticky Note3                          | Sticky Note                   | Explains question splitting                        | —                                    | —                                      | Split out questions.                                                                                                            |
| Sticky Note4                          | Sticky Note                   | Explains Tavily answer retrieval                   | —                                    | —                                      | Get answers using Tavily API. You can replace the community node with direct API calls.                                         |
| Sticky Note5                          | Sticky Note                   | Explains answer aggregation                        | —                                    | —                                      | Add answers to JSON.                                                                                                            |
| Sticky Note6                          | Sticky Note                   | Explains outline aggregation                       | —                                    | —                                      | Turn into one list.                                                                                                             |
| Sticky Note7                          | Sticky Note                   | Explains markdown conversion                       | —                                    | —                                      | Convert JSON into markdown.                                                                                                     |
| Sticky Note8                          | Sticky Note                   | Explains workflow continuation                     | —                                    | —                                      | Continue with the workflow.                                                                                                     |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node: "Enter keyword"**  
   - Type: Form Trigger  
   - Configure form with title: "Outline generator"  
   - Add one required text field labeled "Keyword" with placeholder "What's the keyword you're going for?"  
   - Set webhook ID or use default to receive form submissions.

2. **Add OpenAI Node: "Generate research questions"**  
   - Type: OpenAI (Langchain)  
   - Set model to "gpt-4o"  
   - Configure messages:  
     - User prompt: `Create an outline and research questions for this keyword: {{ $json.Keyword }}`  
     - System prompt: Instructions to output 5-6 sections with related questions in strict JSON format as specified.  
   - Enable JSON output parsing.  
   - Connect input from "Enter keyword" node.  
   - Configure OpenAI credentials (API key).

3. **Add Split Out Node: "Split out list of questions into separate items"**  
   - Type: Split Out  
   - Set field to split: `message.content.outline`  
   - Connect input from "Generate research questions" node.

4. **Add Tavily Node: "Answer research questions"**  
   - Type: Tavily community node  
   - Set query parameter to `={{ $json.question }}`  
   - Options: topic "general", search depth "advanced", include answer "advanced"  
   - Connect input from "Split out list of questions into separate items" node.  
   - Add Tavily API credentials (API key).

5. **Add Set Node: "Add answers to sections"**  
   - Type: Set  
   - Add fields:  
     - `section_title` = `={{ $('Split out list of questions into separate items').item.json.section }}`  
     - `section_question` = `={{ $('Split out list of questions into separate items').item.json.question }}`  
     - `answer` = `={{ $json.answer }}`  
   - Connect input from "Answer research questions" node.

6. **Add Aggregate Node: "Turn into one big item"**  
   - Type: Aggregate  
   - Mode: Aggregate all item data  
   - Destination field: `outline`  
   - Connect input from "Add answers to sections" node.

7. **Add Code Node: "Convert JSON to markdown"**  
   - Type: Code (JavaScript)  
   - Paste the following code:

```javascript
const sections = $input.first().json.outline;

function toMarkdown(data) {
  return data.map(section => {
    return `## ${section.section_title}\n\n**${section.section_question}**\n\n${section.answer}\n`;
  }).join("\n");
}

return [
  {
    json: {
      markdown: toMarkdown(sections)
    }
  }
];
```

   - Connect input from "Turn into one big item" node.

8. **Add NoOp Node: "Continue workflow"**  
   - Type: NoOp  
   - Acts as a continuation point for further steps.  
   - Connect input from "Convert JSON to markdown" node.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                   | Context or Link                                                         |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------|
| Turn your keyword research into a clear, fact-based content outline with this workflow. It splits your keyword into 5-6 subtopics, generates research questions, and uses Tavily to pull accurate answers from real search results. | Overview sticky note in the workflow                                  |
| Get an OpenAI API key and set up your credentials in n8n: https://openai.com/api/                                                                                                                                              | Setup instructions                                                     |
| Sign up for Tavily and obtain an API key: https://www.tavily.com/                                                                                                                                                              | Setup instructions                                                     |
| Tavily API Reference for direct HTTP calls: https://docs.tavily.com/documentation/api-reference/endpoint/search                                                                                                               | Alternative to community node                                          |
| Workflow customization ideas: use Google Docs for input, add AI nodes to generate full articles, export outlines to Google Docs or email via Gmail node                                                                           | Sticky notes 8 and overview                                           |
| The workflow ensures data freshness by combining AI-generated structure with real-time web data, mitigating risks of outdated or hallucinated content from AI models alone.                                                     | General best practice note                                            |

---

**Disclaimer:**  
The provided description and workflow originate exclusively from an automated process created with n8n, an integration and automation tool. This process complies fully with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.