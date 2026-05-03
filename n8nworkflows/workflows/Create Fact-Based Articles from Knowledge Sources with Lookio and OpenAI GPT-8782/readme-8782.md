Create Fact-Based Articles from Knowledge Sources with Lookio and OpenAI GPT

https://n8nworkflows.xyz/workflows/create-fact-based-articles-from-knowledge-sources-with-lookio-and-openai-gpt-8782


# Create Fact-Based Articles from Knowledge Sources with Lookio and OpenAI GPT

### 1. Workflow Overview

This workflow automates the creation of fact-based articles by leveraging user-provided knowledge sources through Lookio and OpenAI's GPT language models. It is designed for content creators who want to produce high-quality, well-researched articles grounded entirely in their own verified knowledge bases.

The workflow is logically divided into four main blocks:

- **1.1 Input Reception:** Collects article title and writing guidelines from the user via a form.
- **1.2 Question Generation:** Uses AI to decompose the article topic into a series of comprehensive, non-overlapping research questions.
- **1.3 Research & Answering:** Queries a Lookio assistant for answers to each generated question, based on connected knowledge sources.
- **1.4 Article Composition:** Uses AI to compose a final article from the aggregated research content, adhering strictly to the initial title and guidelines.

This structured approach ensures that the final article is both factually accurate and stylistically aligned with user requirements, integrating source references when available.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

- **Overview:**  
This block captures the article title and writing guidelines from the user via an interactive form, then prepares these values for downstream processing.

- **Nodes Involved:**  
  - New article form  
  - Prepare form values

- **Node Details:**  
  - **New article form**  
    - Type: Form Trigger  
    - Role: Entry point for user input; triggers workflow execution when the form is submitted.  
    - Configuration: Two required text fields: "Article title" and "Article guidelines" with placeholders to guide input.  
    - Input/Output: No input; outputs JSON with user-submitted form data.  
    - Edge cases: Missing required fields prevent submission; ensure users fill form properly.  
  - **Prepare form values**  
    - Type: Set node  
    - Role: Extracts and renames form fields to `Title` and `Guidelines` for clarity.  
    - Configuration: Assigns `Title` from `Article title`, `Guidelines` from `Article guidelines`.  
    - Input: Output from form trigger.  
    - Output: JSON object with `Title` and `Guidelines`.  
    - Edge cases: No validation beyond form; empty or malformed inputs could affect later AI prompts.

---

#### 2.2 Question Generation

- **Overview:**  
This block generates a structured set of sub-questions that fully cover the article topic, guiding the subsequent research phase.

- **Nodes Involved:**  
  - New content - generate research questions  
  - Structured Output Parser  
  - Split Out Questions  
  - Loop Over Questions

- **Node Details:**  
  - **New content - generate research questions**  
    - Type: Langchain Chain LLM  
    - Role: Uses GPT (gpt-5-mini) to generate 5-8 non-overlapping research questions based on article title and guidelines.  
    - Configuration:  
      - Input prompt includes the article title and guidelines.  
      - Instructions specify question structure: start broad and simple, then become specific and analytical.  
      - Output expected as a JSON array of question objects.  
    - Input: `Title` and `Guidelines` from "Prepare form values".  
    - Output: Raw AI output, parsed by next node.  
    - Edge cases: Possible parsing errors if AI output deviates from JSON format; incomplete or off-topic questions if prompt not clear.  
  - **Structured Output Parser**  
    - Type: Langchain Structured Output Parser  
    - Role: Parses AI output into JSON array for use in workflow.  
    - Configuration: Example JSON schema provided for validation.  
    - Input: AI response from previous node.  
    - Output: Parsed questions as structured data.  
    - Edge cases: Parsing failure if AI response malformed; should handle gracefully or retry.  
  - **Split Out Questions**  
    - Type: Split Out  
    - Role: Splits the array of questions into individual items for separate processing.  
    - Configuration: Splits on `output` field containing question array.  
    - Input: Parsed question array.  
    - Output: Individual question items.  
  - **Loop Over Questions**  
    - Type: Split In Batches  
    - Role: Iterates over each question individually to process research queries.  
    - Configuration: Default batch size (1) to process one question at a time.  
    - Input: Individual questions from Split Out Questions.  
    - Output: One question per iteration for downstream querying.  
    - Edge cases: Large question sets could slow workflow; batching options available.

---

#### 2.3 Research & Answering

- **Overview:**  
This block queries the Lookio assistant for fact-based answers to each research question, collects the results, and formats the question-answer pairs.

- **Nodes Involved:**  
  - Query Lookio Assistant  
  - Format question and answer  
  - Aggregate research content

- **Node Details:**  
  - **Query Lookio Assistant**  
    - Type: HTTP Request  
    - Role: Sends each research question to the Lookio API assistant to retrieve factual answers.  
    - Configuration:  
      - POST request to `https://api.lookio.app/webhook/query`  
      - Body includes `query` (question text), `assistant_id` (user must configure), and `query_mode` set to "flash" for fast response.  
      - Header includes `api_key` for Lookio authentication.  
    - Input: Single question from Loop Over Questions.  
    - Output: Answer text from Lookio assistant.  
    - Edge cases:  
      - Authentication errors if API key or assistant ID missing/incorrect.  
      - Network timeouts or API errors.  
      - Empty or irrelevant answers if knowledge base incomplete.  
    - Notes: User must replace placeholder API key and assistant ID with own credentials.  
  - **Format question and answer**  
    - Type: Set node  
    - Role: Constructs a JSON object containing the original question and the Lookio answer.  
    - Configuration: Assigns two fields: `Question` (question text) and `Answer` (Lookio response).  
    - Input: Answer from Lookio and question from previous iteration.  
    - Output: Structured question-answer pair.  
  - **Aggregate research content**  
    - Type: Aggregate  
    - Role: Collects all question-answer pairs into a single array for final article composition.  
    - Configuration: Aggregates all incoming items into one array under field `Content to leverage`.  
    - Input: Multiple question-answer pairs from Format question and answer.  
    - Output: Single aggregated JSON array for use in article writing.

---

#### 2.4 Article Composition

- **Overview:**  
This block leverages AI to write a full article using the aggregated research content, strictly following the initial title and writing guidelines.

- **Nodes Involved:**  
  - New content - Generate the AI output  
  - Article result

- **Node Details:**  
  - **New content - Generate the AI output**  
    - Type: Langchain Chain LLM  
    - Role: Instructed GPT (gpt-5-chat-latest) to write the article based exclusively on the research content and user guidelines.  
    - Configuration:  
      - Prompt includes article title, guidelines, and the aggregated question-answer research content as JSON.  
      - Instructions emphasize using only the supplied research, including source hyperlinks where available.  
      - Output format requires an H1 title and subheadings throughout the article.  
    - Input: `Title`, `Guidelines` and aggregated `Content to leverage`.  
    - Output: Full article text.  
    - Edge cases: Risk of AI hallucination if research insufficient; ensure clear prompt and high-quality research input.  
  - **Article result**  
    - Type: Set node  
    - Role: Stores the final article text in a field named `Article`.  
    - Configuration: Assigns `Article` field from AI output text.  
    - Input: AI generated article text.  
    - Output: Final JSON object containing article content.

---

### 3. Summary Table

| Node Name                         | Node Type                        | Functional Role                                  | Input Node(s)                  | Output Node(s)                   | Sticky Note                                                                                                       |
|----------------------------------|---------------------------------|-------------------------------------------------|-------------------------------|---------------------------------|-------------------------------------------------------------------------------------------------------------------|
| New article form                 | Form Trigger                    | Collects article title and guidelines from user | None                          | Prepare form values              | Fill in this form to request a new article                                                                        |
| Prepare form values             | Set                             | Formats form data into `Title` and `Guidelines` | New article form              | New content - generate research questions |                                                                                                                   |
| New content - generate research questions | Langchain Chain LLM             | Generates sub-questions based on title/guidelines| Prepare form values           | Split Out Questions             | ## Breaking down the topic into sub-questions                                                                     |
| Structured Output Parser        | Langchain Structured Output Parser | Parses AI-generated JSON questions                | New content - generate research questions | New content - generate research questions (parser output) |                                                                                                                   |
| Split Out Questions             | Split Out                      | Splits question array into individual questions   | Structured Output Parser      | Loop Over Questions             | ## Answering each sub-question one by one with Lookio                                                             |
| Loop Over Questions             | Split In Batches               | Iterates over each question for querying          | Split Out Questions           | Aggregate research content, Query Lookio Assistant |                                                                                                                   |
| Query Lookio Assistant          | HTTP Request                  | Fetches answers from Lookio assistant API         | Loop Over Questions           | Format question and answer      | Connect your lookio.app credentials & replace your assistant ID                                                   |
| Format question and answer      | Set                             | Combines each question with its answer             | Query Lookio Assistant        | Loop Over Questions             |                                                                                                                   |
| Aggregate research content      | Aggregate                      | Collects all Q&A pairs for article composition     | Loop Over Questions           | New content - Generate the AI output |                                                                                                                   |
| New content - Generate the AI output | Langchain Chain LLM             | Writes final article based on research and inputs | Aggregate research content, Prepare form values | Article result                 | ## AI step writing the final article based on the research and initial request                                    |
| Article result                 | Set                             | Stores final article content                        | New content - Generate the AI output | None                          |                                                                                                                   |
| Sticky Note (multiple nodes)    | Sticky Note                    | Various notes on usage and setup                    | None                          | None                           | See detailed notes in 5. General Notes & Resources                                                                |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "New article form" node**  
   - Type: Form Trigger  
   - Configure form fields:  
     - "Article title" (required, text, placeholder: "10 ways to do Influencer Marketing in 2025")  
     - "Article guidelines" (required, text, placeholder: "Promote xyz and write in British English...")  
   - Set a form description: "Fill in this form to trigger the generation of a new article."

2. **Create "Prepare form values" node**  
   - Type: Set  
   - Assign two fields:  
     - `Title` = `{{$json["Article title"]}}`  
     - `Guidelines` = `{{$json["Article guidelines"]}}`  
   - Connect "New article form" output to this node.

3. **Create "New content - generate research questions" node**  
   - Type: Langchain Chain LLM  
   - Choose model: `gpt-5-mini`  
   - Prompt template:  
     ```
     Content title:  {{ $json.Title }}

     Article guidelines: {{ $json.Guidelines }}

     You will receive a content title and an angle. Return 5–8 non-overlapping questions in JSON array format that cover everything needed to write excellent content as it breaks down the topic into sub-questions.

     Guidelines:  
     - Start with simple, short broad questions (e.g., What is X?, Why is X important?, How to do X?).  
     - Then move into more specific, advanced, or analytical questions.  
     - Ensure questions together form a complete coverage of the topic.

     ## Output format:

     You'll return the questions in such a JSON ARRAY:

     [
       {
         "question": "..."
       },
       ...
     ]
     ```
   - Connect "Prepare form values" output here.

4. **Create "Structured Output Parser" node**  
   - Type: Langchain Structured Output Parser  
   - Provide example JSON schema matching the expected array of question objects.  
   - Connect "New content - generate research questions" AI output to this node's input parser.

5. **Create "Split Out Questions" node**  
   - Type: Split Out  
   - Configure to split the array field containing questions (`output`).  
   - Connect "Structured Output Parser" output here.

6. **Create "Loop Over Questions" node**  
   - Type: Split In Batches  
   - Default batch size (1) to process questions sequentially.  
   - Connect "Split Out Questions" output here.

7. **Create "Query Lookio Assistant" node**  
   - Type: HTTP Request  
   - Method: POST  
   - URL: `https://api.lookio.app/webhook/query`  
   - Body Parameters (JSON):  
     - `query`: `={{ $json.question }}`  
     - `assistant_id`: `<your-assistant-id>` (replace with your valid Lookio assistant ID)  
     - `query_mode`: `"flash"`  
   - Header Parameters:  
     - `api_key`: `<your-lookio-api-key>` (replace with your Lookio API key)  
   - Connect "Loop Over Questions" output here.

8. **Create "Format question and answer" node**  
   - Type: Set  
   - Assign fields:  
     - `Question` = `={{ $('Loop Over Questions').item.json.question }}`  
     - `Answer` = `={{ $json.Output }}` (response from Lookio)  
   - Connect "Query Lookio Assistant" output here.

9. **Connect "Format question and answer" output back to "Loop Over Questions" input**  
   - This creates a loop to process all questions individually.

10. **Create "Aggregate research content" node**  
    - Type: Aggregate  
    - Aggregate all question-answer pairs into a single array named `Content to leverage`.  
    - Connect one output from "Loop Over Questions" (aggregate branch) here.

11. **Create "New content - Generate the AI output" node**  
    - Type: Langchain Chain LLM  
    - Choose model: `gpt-5-chat-latest`  
    - Prompt template:  
      ```
      Article title:

      {{ $('Prepare form values').first().json.Title }}

      Article guidelines:

      {{ $('Prepare form values').first().json.Guidelines }}


      Content to leverage:

      This Q&A research provides high-quality knowledge, insights, and sources for your content. Be sure to include source links in your output whenever a source was used.

      {{ JSON.stringify($json['Content to leverage'], null, 2) }}

      # Role

      Your role is to write an article based on the user request.

      # How to write good articles

      You excel at writing articles that deliver value, are concise, feel human-written, and avoid typical AI clichés.

      # Output format

      Output only the full article.

      * Begin with a `# H1` title.
      * Use subheadings throughout the article.
      ```
    - Connect "Aggregate research content" output here.

12. **Create "Article result" node**  
    - Type: Set  
    - Assign field `Article` = `={{ $json.text }}` (the AI-generated article text).  
    - Connect "New content - Generate the AI output" output here.

13. **Configure Credentials**  
    - Add OpenAI credentials named "Duv's OpenAI" (or your own) with API key for GPT nodes.  
    - Add Lookio API key and assistant ID in "Query Lookio Assistant" node (replace placeholders).

14. **Test the workflow**  
    - Submit the form with an article title and guidelines.  
    - Verify generation of sub-questions.  
    - Confirm Lookio returns answers for each question.  
    - Check final article output matches input guidelines and includes research-based content.

---

### 5. General Notes & Resources

| Note Content                                                                                                                         | Context or Link                                                               |
|--------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------|
| This workflow automates the creation of AI-written articles grounded in user knowledge bases via Lookio and OpenAI GPT.              | Overview and project purpose                                                  |
| Connect your Lookio credentials (API key and Assistant ID) before running the workflow to enable research queries.                   | Sticky Note at "Query Lookio Assistant" node                                 |
| The AI planner decomposes the topic into questions, queries your knowledge base, then writes the article based solely on verified facts. | Sticky Note summarizing workflow logic                                       |
| For best results, build your assistant in Lookio with comprehensive, high-quality knowledge sources connected to it.                | Instructions in Sticky Notes and project description                         |
| OpenAI GPT model versions used: gpt-5-mini for question generation, gpt-5-chat-latest for article writing.                           | Node configurations                                                           |
| This workflow was built by Guillaume Duvernay and is a template for fact-based AI article generation.                                | Project credits                                                               |
| For more details and updates, visit Guillaume’s blog or Lookio documentation (links not embedded in workflow but recommended).       | Suggested external resources                                                  |

---

**Disclaimer:**  
The text processed and generated by this workflow is produced exclusively by an automated n8n workflow respecting all content policies. It contains no illegal, offensive, or protected elements. All data manipulated is legal and public.