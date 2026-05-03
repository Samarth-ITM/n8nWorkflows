Create SEO-Optimized Blog Articles with GPT-4o Mini's Multi-Stage Generation

https://n8nworkflows.xyz/workflows/create-seo-optimized-blog-articles-with-gpt-4o-mini-s-multi-stage-generation-10813


# Create SEO-Optimized Blog Articles with GPT-4o Mini's Multi-Stage Generation

---

### 1. Workflow Overview

This n8n workflow automates the creation of SEO-optimized blog articles using GPT-4o Mini’s multi-stage text generation capabilities. It is designed primarily for content creators, marketers, and SEO professionals who want to generate detailed, well-structured articles with SEO-friendly readability automatically.

The workflow is logically divided into three major blocks:

- **1.1 Input Reception and Outline Generation:** Captures user input for the blog topic, preferred language, number of outline items, and keyword, then generates a structured article outline using an AI agent.

- **1.2 Looping over Subtopics for Detailed Content Generation:** Iterates through each subtopic of the outline, enriching it into detailed, SEO-friendly paragraphs using a memory-enhanced AI agent that respects YOAST SEO readability standards.

- **1.3 Aggregation and Output Formatting:** Aggregates generated paragraphs, formats the content into Markdown and HTML versions suitable for blog publishing, and prepares structured JSON output for further workflow extensions (e.g., posting to WordPress).

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Outline Generation

**Overview:**  
This block gathers user parameters, then uses an AI agent to generate a creative, structured outline for the blog article. The output is a JSON object containing the main topic and an array of subtopics with details.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Input Topic Plans (Set)  
- OpenAI Chat Model (GPT-4o Mini)  
- Generate Outline (Langchain Agent)  
- Structured Output Parser  
- Get Subtopic Array (For Looping)

**Node Details:**

- **When clicking ‘Execute workflow’**  
  - *Type:* Manual trigger  
  - *Role:* Entry point to start the workflow manually.  
  - *Input/Output:* No input; outputs trigger data to “Input Topic Plans.”  
  - *Edge Cases:* None significant; manual start implies user presence.

- **Input Topic Plans**  
  - *Type:* Set  
  - *Role:* Defines key input parameters: Topic, number of outline items, language, and SEO keyword.  
  - *Configuration:* Default placeholders for user to replace:  
    - Topic: "Input your topic Here"  
    - How many outline?: 3  
    - Language: "Input the language here (ex: English/Indonesia/Arabic/etc)"  
    - Keyword: "Put your keyword here"  
  - *Output:* JSON object with these fields.  
  - *Edge Cases:* Missing or invalid inputs may cause poor outline generation or empty results.

- **OpenAI Chat Model**  
  - *Type:* Langchain OpenAI Chat Model node configured to use GPT-4o Mini model.  
  - *Role:* Provides the AI language model interface for “Generate Outline.”  
  - *Credentials:* Uses OpenAI API key configured in n8n credentials.  
  - *Edge Cases:* API key errors, rate limits, network timeouts.

- **Generate Outline**  
  - *Type:* Langchain Agent  
  - *Role:* Generates a JSON structured article outline with topic, subtopics, and details.  
  - *Prompt:* Requests maximum outline items (from input), in specified language, with a JSON output schema. Starts with Introduction and ends with Conclusion.  
  - *System Message:* Defines AI as an expert fluent in Bahasa Indonesia with broad knowledge.  
  - *Output Parser:* Enabled with a JSON schema example to enforce structured output.  
  - *Input Connections:* Receives AI model from “OpenAI Chat Model” and input from “Input Topic Plans.”  
  - *Edge Cases:* Parsing errors if output is malformed; AI hallucination or irrelevant outlines.

- **Structured Output Parser**  
  - *Type:* Langchain Structured Output Parser  
  - *Role:* Parses the AI-generated text into the defined JSON schema.  
  - *Configuration:* JSON schema example includes topic string and subtopics array with subtopic and details strings.  
  - *Input:* Output from “Generate Outline.”  
  - *Output:* Parsed JSON for further processing.  
  - *Edge Cases:* Parsing failures if AI output does not match schema; fallback or error handling needed.

- **Get Subtopic Array (For Looping)**  
  - *Type:* Code (JavaScript)  
  - *Role:* Extracts the “subtopics” array from the parsed JSON outline to prepare for looping.  
  - *Code:* Returns the subtopics array from the first JSON input item’s output.  
  - *Input:* Parsed outline JSON.  
  - *Output:* Array of subtopic objects for batch processing.  
  - *Edge Cases:* Empty or missing subtopics array; may cause downstream loop to process zero items.

---

#### 2.2 Looping over Subtopics for Detailed Content Generation

**Overview:**  
This block loops over each subtopic extracted from the outline, generating detailed, SEO-optimized paragraphs per subtopic using an AI agent with a simple memory buffer to maintain context relevance.

**Nodes Involved:**  
- Loop Over Items (SplitInBatches)  
- AI Agent1 (Langchain Agent)  
- OpenAI Chat Model1  
- Simple Memory  
- Create New Array

**Node Details:**

- **Loop Over Items**  
  - *Type:* SplitInBatches  
  - *Role:* Iterates over the array of subtopic objects, processing one or more items per batch (default batch size).  
  - *Input:* Array of subtopics from “Get Subtopic Array (For Looping).”  
  - *Output:* Individual subtopic items to “AI Agent1” and aggregation node.  
  - *Edge Cases:* Empty input arrays; batch size limits; potential processing delays.

- **AI Agent1**  
  - *Type:* Langchain Agent  
  - *Role:* Generates a detailed paragraph for each subtopic based on subtopic title and details, following strict SEO and readability guidelines.  
  - *Prompt Highlights:*  
    - Write max 1 paragraph per subtopic.  
    - No numbering or promotional style.  
    - Use the keyword provided in “Input Topic Plans.”  
    - Maintain >75% sentences with ≤20 words for YOAST SEO readability.  
    - Use 1-2 transition words (examples provided).  
    - Write in user-specified language.  
  - *System Message:* AI is an expert writer providing dense detail and examples.  
  - *Input:* Receives subtopic and details from “Loop Over Items,” and keyword/language from “Input Topic Plans.”  
  - *Output:* Generated paragraph per subtopic.  
  - *Edge Cases:* AI may generate off-topic content, fail to meet SEO style, or produce empty outputs.

- **OpenAI Chat Model1**  
  - *Type:* Langchain OpenAI Chat Model  
  - *Role:* Provides GPT-4o Mini model interface for “AI Agent1.”  
  - *Credentials:* Same OpenAI API key as before.  
  - *Edge Cases:* API errors, rate limits.

- **Simple Memory**  
  - *Type:* Langchain Memory Buffer Window  
  - *Role:* Maintains a short-term memory buffer keyed by the current subtopic details to keep AI contextually relevant across iterations.  
  - *Session Key:* Derived dynamically from subtopic details.  
  - *Input:* Receives memory input/output from “AI Agent1.”  
  - *Edge Cases:* Memory size limits; key collisions; loss of context if session key is malformed.

- **Create New Array**  
  - *Type:* Code (JavaScript)  
  - *Role:* Loops over items and adds or adjusts fields as needed; specifically adds the subtopic field from the first item to each item’s JSON for consistency.  
  - *Code Notes:* The code appears to redundantly set subtopic to the first subtopic’s name for all items, which may be unintended.  
  - *Input:* Output of “AI Agent1.”  
  - *Output:* Prepared array forwarded to the next block.  
  - *Edge Cases:* Potential logic error causing all subtopics to be identical in output.

---

#### 2.3 Aggregation and Output Formatting

**Overview:**  
This block aggregates all generated paragraphs and formats the final article content into Markdown and HTML formats, suitable for SEO-friendly blog publication or API consumption.

**Nodes Involved:**  
- Aggregate  
- JSON HTML (Code)

**Node Details:**

- **Aggregate**  
  - *Type:* Aggregate  
  - *Role:* Aggregates all batch items from the loop into a single dataset for final processing.  
  - *Configuration:* Uses “aggregate all item data” option.  
  - *Input:* Output from “Loop Over Items” (main output 0).  
  - *Output:* Single array containing all generated paragraphs.  
  - *Edge Cases:* Large data sets may cause performance issues; empty input results in empty aggregation.

- **JSON HTML**  
  - *Type:* Code (JavaScript)  
  - *Role:* Transforms aggregated JSON data into two formats: Markdown and HTML.  
  - *Code Logic:*  
    - Extracts article title from the original outline topic.  
    - Iterates through article sections: first item is output only; subsequent items prepend subtopic headings.  
    - Converts paragraphs into `<p>` tags for HTML and uses Markdown headers `##` for subtopics.  
    - Trims excess line breaks.  
    - Returns JSON with `title`, `content_markdown`, and `content_html`.  
  - *Input:* Aggregated data from “Aggregate.” Also references the “Generate Outline” node for the title.  
  - *Output:* Final formatted content JSON for downstream usage.  
  - *Edge Cases:* Missing or malformed data may cause output formatting issues.

---

### 3. Summary Table

| Node Name                   | Node Type                          | Functional Role                          | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                      |
|-----------------------------|----------------------------------|----------------------------------------|--------------------------------|-------------------------------|------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                   | Workflow entry point                    | -                              | Input Topic Plans              |                                                                                                |
| Input Topic Plans            | Set                              | Defines user input parameters           | When clicking ‘Execute workflow’ | Generate Outline               |                                                                                                |
| OpenAI Chat Model            | Langchain OpenAI Chat Model       | Provides GPT-4o Mini AI model           | -                              | Generate Outline               |                                                                                                |
| Generate Outline             | Langchain Agent                  | Generates structured article outline    | Input Topic Plans, OpenAI Chat Model, Structured Output Parser | Get Subtopic Array (For Looping) | ## 1. Creating Outline                                                                         |
| Structured Output Parser     | Langchain Structured Output Parser | Parses AI output into JSON structure    | Generate Outline               | Generate Outline               |                                                                                                |
| Get Subtopic Array (For Looping) | Code (JavaScript)                | Extracts subtopics array for looping    | Generate Outline               | Loop Over Items               |                                                                                                |
| Loop Over Items              | SplitInBatches                   | Iterates over subtopics                  | Get Subtopic Array             | Aggregate, AI Agent1           | ## 2. Loop the Outline and Generate Paragraph<br>We also using simple memory to make sure the loop still relevant |
| AI Agent1                   | Langchain Agent                  | Generates detailed paragraph per subtopic | Loop Over Items, OpenAI Chat Model1, Simple Memory | Create New Array              |                                                                                                |
| OpenAI Chat Model1           | Langchain OpenAI Chat Model       | Provides GPT-4o Mini model for AI Agent1 | -                              | AI Agent1                    |                                                                                                |
| Simple Memory               | Langchain Memory Buffer Window   | Maintains context memory per subtopic   | AI Agent1                     | AI Agent1                    |                                                                                                |
| Create New Array             | Code (JavaScript)                | Adds/adjusts subtopic field in each item | AI Agent1                     | Loop Over Items               |                                                                                                |
| Aggregate                   | Aggregate                       | Aggregates all loop outputs into single array | Loop Over Items               | JSON HTML                   | ## 3.Result become JSON with Markdown and HTML format<br>Just continue this nodes with your purpose |
| JSON HTML                   | Code (JavaScript)                | Formats aggregated content to Markdown and HTML | Aggregate, Generate Outline  | -                             |                                                                                                |
| Sticky Note                 | Sticky Note                     | Article Generator overview and setup instructions | -                              | -                             | ## Article Generator SEO Friendly<br>• Generates a structured outline from your topic using an AI Agent.<br>• Loops through each subtopic and expands it into a detailed, SEO-friendly paragraph.<br>• Combines everything into two final outputs. A clean JSON object and a Markdown version.<br>• Content is tested in Yoast SEO and achieves Good readability.<br>• Uses GPT-4o Mini by default. Token usage is usually 2000 to 3000 depending on the number of outline items.<br><br>### Set up steps<br>• Prepare your OpenAI API Key.<br>• Adjust the topic, outline count, language, and keyword as needed in teh *Edit Fields*.<br>• Change the model anytime if you want different quality or token usage. |
| Sticky Note1                | Sticky Note                     | Label for outline creation block        | -                              | -                             | ## 1. Creating Outline                                                                         |
| Sticky Note2                | Sticky Note                     | Label for looping & paragraph generation | -                              | -                             | ## 2. Loop the Outline and Generate Paragraph<br>We also using simple memory to make sure the loop still relevant |
| Sticky Note3                | Sticky Note                     | Label for final result formatting block | -                              | -                             | ## 3.Result become JSON with Markdown and HTML format<br>Just continue this nodes with your purpose |
| Sticky Note4                | Sticky Note                     | Workflow creator profile and contact info | -                              | -                             | ## Workflow Creator Profile – Pake.AI<br>**Pake.AI** is an AI Enabler from Indonesia, committed to helping creators, entrepreneurs, and businesses automate their operations through practical and accessible AI solutions.<br><br>This workflow is shared **for free** to support the community and accelerate AI adoption.<br><br>If you have any questions about this workflow or want to explore more automation tools, feel free to contact us:<br>🌐 **https://pake.ai** |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Manual Trigger Node**  
   - Type: Manual Trigger  
   - Purpose: Start workflow on demand.

2. **Create the ‘Input Topic Plans’ Set Node**  
   - Type: Set  
   - Configure fields:  
     - `Topic` (string): Default placeholder “Input your topic Here”  
     - `How many outline?` (number): Default 3  
     - `Language` (string): Default “Input the language here (ex: English/Indonesia/Arabic/etc)”  
     - `Keyword` (string): Default “Put your keyword here”  
   - Connect output of Manual Trigger to this node.

3. **Create OpenAI Chat Model Node**  
   - Type: Langchain OpenAI Chat Model  
   - Model: GPT-4o Mini  
   - Credentials: Configure with your OpenAI API key credentials.  
   - No special options required.

4. **Create ‘Generate Outline’ Langchain Agent Node**  
   - Type: Langchain Agent  
   - Connect input from ‘Input Topic Plans’ node and OpenAI Chat Model node (as AI Language Model).  
   - Configuration:  
     - Prompt (Text):  
       ```
       Write a creative outline max {{ $json["How many outline?"] }} item on {{ $json.Topic }} in {{ $json.Language }} language.

       Output a JSON object with the following structure: { "topic": "string", "subtopics": [ { "subtopic": "string", "details": "string" } ] } Start the subtopic with an Introduction and end with a Conclusion.
       ```
     - System Message:  
       “You are a wise expert writer who fluent with Bahasa Indonesia and has broad knowledge ranging from general overview to niche subtopics, including rarely known small details.”  
     - Enable Output Parser and set to Structured Output Parser node (to create next).

5. **Create Structured Output Parser Node**  
   - Type: Langchain Structured Output Parser  
   - JSON Schema Example:  
     ```json
     { "topic": "string", "subtopics": [ { "subtopic": "string", "details": "string" } ] }
     ```  
   - Connect input from ‘Generate Outline’ node.

6. **Create ‘Get Subtopic Array (For Looping)’ Code Node**  
   - Type: Code (JavaScript)  
   - Code:  
     ```js
     return $input.first().json.output.subtopics;
     ```  
   - Connect input from ‘Generate Outline’ node (parsed output).

7. **Create ‘Loop Over Items’ Node (SplitInBatches)**  
   - Type: SplitInBatches  
   - Connect input from ‘Get Subtopic Array (For Looping).’  
   - Default batch size is OK.

8. **Create OpenAI Chat Model1 Node**  
   - Duplicate of step 3 (for AI Agent1 usage).  
   - Use same GPT-4o Mini model and credentials.

9. **Create ‘Simple Memory’ Node**  
   - Type: Langchain Memory Buffer Window  
   - Session Key: Set to `={{ $('Loop Over Items').item.json.details }}`  
   - Session Id Type: customKey  
   - Connect memory input/output to ‘AI Agent1’ node (to be created next).

10. **Create ‘AI Agent1’ Node (Langchain Agent)**  
    - Connect input from ‘Loop Over Items’ node (main output 0), OpenAI Chat Model1 (AI Language Model), and Simple Memory (memory).  
    - Prompt (Text):  
      ```
      The outline refers to parts of a conversation.
      This is the article section that covers these topics:

      Subtopic: {{ $json.subtopic }}
      Details: {{ $json.details }}

      Write a maximum of 1 paragraph.
      Do not number any sections.
      Include a lot of detail.

      Mandatory Rules

      1. Do not add numbering in any part.
      2. Write with detail and dense information.
      3. Ensure more than 75 percent of sentences contain a maximum of 20 words to meet YOAST SEO readability standards.
      4. Ensure the keyword "{{ $('Input Topic Plans').item.json.Keyword }}" appears in the sentences.
      5. Use clear and active sentences.
      6. Use 1 or 2 transition words recognized by YOAST. Examples: so, therefore, however, besides that, then, after that, for example.
      7. Avoid emojis, excessive punctuation, and promotional style.
      8. Do not use em dashes to keep the writing natural and not AI generated.
      9. Write in {{ $('Input Topic Plans').item.json.Language }} Language
      ```  
    - System Message:  
      “You are a wise expert writer who shares an abundance of detail and examples, covering niche subtopics and rarely known small details.”  
    - PromptType: Define

11. **Create ‘Create New Array’ Code Node**  
    - Type: Code (JavaScript)  
    - Code:  
      ```js
      for (const item of $input.all()) {
        item.json.subtopic = $('Loop Over Items').first().json.subtopic;
      }
      return $input.all();
      ```  
    - Connect input from ‘AI Agent1’ node.

12. **Connect ‘Create New Array’ output back to ‘Loop Over Items’ input**  
    - This completes the iteration loop.

13. **Create ‘Aggregate’ Node**  
    - Type: Aggregate  
    - Aggregate: Aggregate all item data.  
    - Connect input from ‘Loop Over Items’ main output 0 (aggregated data).

14. **Create ‘JSON HTML’ Code Node**  
    - Type: Code (JavaScript)  
    - Code (summary):  
      - Extracts title from ‘Generate Outline’ node.  
      - Converts aggregated array to Markdown and HTML:  
        - First item output only.  
        - Subsequent items with subtopic headers and paragraph tags.  
      - Returns JSON with `title`, `content_markdown`, `content_html`.  
    - Connect input from ‘Aggregate’ node and reference ‘Generate Outline’ for title.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                     | Context or Link                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------|
| Workflow created by **Pake.AI**, Indonesia-based AI Enabler supporting creators and businesses with AI automation solutions. Shared freely to accelerate AI adoption. For questions or more automation tools visit https://pake.ai | Creator Profile & Contact Information |
| The workflow uses GPT-4o Mini model by default, balancing token usage (~2000-3000 tokens depending on outline size) and content quality.                                                                                       | Model Choice                          |
| The generated content is structured and optimized for Yoast SEO with readability constraints enforced in AI prompts (e.g., sentence length, transition words).                                                                 | SEO Optimization                      |
| Editable parameters include topic, number of outline items, language, and keyword. Adjust these in the “Input Topic Plans” node before execution.                                                                               | User Customization                    |
| The output includes both Markdown and HTML formats, facilitating easy integration with WordPress or other CMS platforms supporting REST API content ingestion.                                                                  | Output Formatting                     |

---

**Disclaimer:** The provided text is exclusively generated from an automated n8n workflow. It strictly adheres to content policies and contains no illegal, offensive, or protected elements. All data processed are legal and public.

---