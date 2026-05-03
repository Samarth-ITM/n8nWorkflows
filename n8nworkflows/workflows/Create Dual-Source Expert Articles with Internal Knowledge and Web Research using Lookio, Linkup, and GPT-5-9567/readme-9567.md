Create Dual-Source Expert Articles with Internal Knowledge and Web Research using Lookio, Linkup, and GPT-5

https://n8nworkflows.xyz/workflows/create-dual-source-expert-articles-with-internal-knowledge-and-web-research-using-lookio--linkup--and-gpt-5-9567


# Create Dual-Source Expert Articles with Internal Knowledge and Web Research using Lookio, Linkup, and GPT-5

---

### 1. Workflow Overview

This workflow automates the creation of expert-level articles grounded in both internal knowledge bases and web research. Its primary use case is to generate high-quality content by decomposing an article topic into research questions, gathering insights from two sources—Lookio (internal knowledge) and Linkup (web search)—and then synthesizing these insights into a coherent article using advanced AI language models.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception and Preparation:** Collects article title and guidelines from a user form and prepares these inputs for processing.
- **1.2 Question Generation:** Uses an AI language model to generate a comprehensive set of research questions that cover the article topic.
- **1.3 Research and Insight Gathering:** Iteratively queries the Lookio Assistant for internal knowledge and Linkup API for web insights for each question.
- **1.4 Research Aggregation:** Aggregates all the gathered research insights into a structured dataset.
- **1.5 Article Writing:** Uses a powerful AI language model (GPT-5 chat) to write the final article based on the aggregated research.
- **1.6 Output Preparation:** Stores the final article text in a dedicated node for output or further use.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Preparation

- **Overview:** This block captures user input via a form and prepares the data for subsequent processing.
- **Nodes Involved:**  
  - New article form  
  - Prepare form values

- **Node Details:**

  - **New article form**  
    - *Type/Role:* Form Trigger — Entry point to collect user inputs.  
    - *Configuration:*  
      - Form fields: "Article title" (required), "Article guidelines" (required).  
      - Webhook enabled to trigger the workflow.  
    - *Input/Output:* No inputs; outputs JSON with user-submitted title and guidelines.  
    - *Edge Cases:* User submits incomplete form (required fields prevent this). Network errors in webhook delivery.  
    - *Sticky Notes:* Explains usage as the form to request a new article.

  - **Prepare form values**  
    - *Type/Role:* Set node — Normalizes and assigns form field values to standardized variables `Title` and `Guidelines`.  
    - *Configuration:* Copies `Article title` and `Article guidelines` from form JSON to `Title` and `Guidelines`.  
    - *Input:* Output from form node.  
    - *Output:* JSON with `Title` and `Guidelines` for downstream nodes.  
    - *Edge Cases:* Expression failures if input fields missing (unlikely due to required form fields).

#### 2.2 Question Generation

- **Overview:** Decomposes the article topic into 5-8 non-overlapping research questions covering the subject fully, starting broad and progressively becoming more specific.
- **Nodes Involved:**  
  - New content - generate research questions  
  - Structured Output Parser  
  - Split Out Questions

- **Node Details:**

  - **New content - generate research questions**  
    - *Type/Role:* LangChain LLM node (AI prompt) — Generates research questions from the article title and guidelines.  
    - *Configuration:*  
      - Prompt includes article title and guidelines.  
      - Instructions specify output as JSON array of questions.  
      - Uses GPT-5 mini model for economical question generation.  
    - *Input:* JSON containing `Title` and `Guidelines`.  
    - *Output:* Raw AI output with questions in JSON array format.  
    - *Edge Cases:* AI could produce malformed JSON; mitigated by the structured output parser.  
    - *Sticky Notes:* Related note on breaking down topic into sub-questions.

  - **Structured Output Parser**  
    - *Type/Role:* LangChain output parser — Parses the AI-generated JSON string into structured JSON objects.  
    - *Configuration:* Uses JSON schema example to enforce parsing.  
    - *Input:* Raw AI output from previous node.  
    - *Output:* Parsed array of question objects.  
    - *Edge Cases:* Parsing failure if AI output malformed or incomplete.

  - **Split Out Questions**  
    - *Type/Role:* Split Out — Separates the array of questions into individual items for iterative processing.  
    - *Input:* Parsed array of questions.  
    - *Output:* Single question per item downstream.  
    - *Edge Cases:* Empty array would halt downstream processing.

#### 2.3 Research and Insight Gathering

- **Overview:** For each question, the workflow sequentially queries the internal knowledge base (Lookio Assistant) and the web (Linkup) to gather complementary insights.
- **Nodes Involved:**  
  - Loop Over Questions  
  - Query Lookio Assistant  
  - Query Linkup for AI web-search  
  - Format question and insights

- **Node Details:**

  - **Loop Over Questions**  
    - *Type/Role:* Split In Batches — Iterates over each question individually for research.  
    - *Input:* Individual question items.  
    - *Output:* Looped question passed to Lookio query node and research aggregation.  
    - *Edge Cases:* Batch size defaults to 1; failure to loop stops research.

  - **Query Lookio Assistant**  
    - *Type/Role:* HTTP Request — Sends the current question to Lookio API to query internal knowledge base.  
    - *Configuration:*  
      - POST to `https://api.lookio.app/webhook/query`.  
      - Body includes the question, assistant ID, and query mode "flash".  
      - API key passed via header.  
      - Requires user to input `<YOUR ASSISTANT ID>` and `<YOUR API KEY>`.  
    - *Input:* Current question from loop.  
    - *Output:* JSON with internal knowledge insights.  
    - *Edge Cases:* Authentication errors if keys missing/invalid; network timeouts; malformed responses.  
    - *Sticky Notes:* Guidance on setting Lookio credentials and assistant ID.

  - **Query Linkup for AI web-search**  
    - *Type/Role:* HTTP Request — Queries Linkup API to fetch 5 specific, meaningful web insights per question.  
    - *Configuration:*  
      - POST to `https://api.linkup.so/v1/search`.  
      - Body contains query request for 5 concrete insights with source URLs.  
      - Uses generic HTTP Bearer Authentication with stored Linkup credentials.  
      - Structured output schema requests array of insights including insight text, URL, and title.  
      - Image inclusion disabled.  
    - *Input:* Output from Lookio node; current question passed in body.  
    - *Output:* JSON array of web insights.  
    - *Edge Cases:* Auth failure if credentials missing; API rate limits; malformed output.  
    - *Sticky Notes:* Reminds to connect Linkup credentials.

  - **Format question and insights**  
    - *Type/Role:* Set node — Consolidates question, internal knowledge insights, and web insights into a single JSON object.  
    - *Configuration:* Assigns:  
      - `Question` from current loop question.  
      - `Internal knowledge insights` from Lookio response output.  
      - `Web insights` from Linkup response.  
    - *Input:* Outputs of Lookio and Linkup queries.  
    - *Output:* Structured research data for each question.  
    - *Edge Cases:* Missing insights may cause incomplete data downstream.

#### 2.4 Research Aggregation

- **Overview:** Aggregates all individual question research results into a single dataset to be used for article generation.
- **Nodes Involved:**  
  - Aggregate full research

- **Node Details:**

  - **Aggregate full research**  
    - *Type/Role:* Aggregate — Combines all looped research items into one array under `Content to leverage`.  
    - *Input:* Stream of formatted research per question.  
    - *Output:* Single JSON object with aggregated content.  
    - *Edge Cases:* Empty input if no research gathered; aggregation failure in case of data corruption.

#### 2.5 Article Writing

- **Overview:** Generates the full article text based solely on the aggregated research content and initial user inputs.
- **Nodes Involved:**  
  - New content - Generate the AI output  
  - GPT 5 chat  
  - Article result

- **Node Details:**

  - **New content - Generate the AI output**  
    - *Type/Role:* LangChain LLM node (AI prompt) — Takes article title, guidelines, and aggregated research to generate the article.  
    - *Configuration:*  
      - Prompt includes title, guidelines, and JSON stringified research content.  
      - Instructions emphasize writing a valuable, concise human-like article with embedded source links.  
      - Uses GPT-5 chat latest model for high-quality writing.  
    - *Input:* Aggregated research content plus user inputs.  
    - *Output:* Full article text.  
    - *Edge Cases:* AI hallucination risk minimized by grounding in research; prompt failure or API errors possible.  
    - *Sticky Notes:* Notes on AI writing step based on research.

  - **GPT 5 chat**  
    - *Type/Role:* Language model node providing the AI engine for content generation.  
    - *Credentials:* OpenAI API key (Duv’s OpenAI).  
    - *Input/Output:* Connected as LLM engine for the generation node.

  - **Article result**  
    - *Type/Role:* Set node — Stores final article text under the field `Article`.  
    - *Input:* Output from AI generation node.  
    - *Output:* Final article JSON object.  
    - *Edge Cases:* Text field could be empty if AI returns no output.

---

### 3. Summary Table

| Node Name                      | Node Type                          | Functional Role                                     | Input Node(s)                  | Output Node(s)                       | Sticky Note                                                                                      |
|--------------------------------|----------------------------------|----------------------------------------------------|-------------------------------|------------------------------------|------------------------------------------------------------------------------------------------|
| New article form               | Form Trigger                     | Receives article title and guidelines from user   | -                             | Prepare form values                 | ## Fill in this form to request a new article                                                  |
| Prepare form values            | Set                              | Prepares and normalizes form inputs                 | New article form              | New content - generate research questions |                                                                                                |
| New content - generate research questions | LangChain LLM (chainLlm)         | Generates research questions from title & guidelines | Prepare form values           | Split Out Questions                 | ## Breaking down the topic into sub-questions                                                  |
| Structured Output Parser       | LangChain Output Parser           | Parses AI JSON output into structured questions     | New content - generate research questions | New content - generate research questions |                                                                                                |
| Split Out Questions            | Split Out                        | Splits questions array into individual questions    | Structured Output Parser      | Loop Over Questions                | ## Answering each sub-question one by one with Lookio for internal knowledge insights & Linkup for web-insights |
| Loop Over Questions            | Split In Batches                 | Iterates over each question for research            | Split Out Questions           | Aggregate full research, Query Lookio Assistant |                                                                                                |
| Query Lookio Assistant         | HTTP Request                    | Queries internal knowledge base (Lookio) for insights | Loop Over Questions           | Query Linkup for AI web-search    | Connect your [lookio.app](https://www.lookio.app) credentials & set the ID of the Assistant to query |
| Query Linkup for AI web-search | HTTP Request                    | Queries web search API (Linkup) for web insights    | Query Lookio Assistant        | Format question and insights       | Connect your linkup.so credentials (adding your API key in the header or using "generic credentials") |
| Format question and insights   | Set                              | Combines question, internal, and web insights       | Query Linkup for AI web-search | Loop Over Questions                |                                                                                                |
| Aggregate full research        | Aggregate                       | Aggregates research insights for all questions      | Loop Over Questions           | New content - Generate the AI output |                                                                                                |
| New content - Generate the AI output | LangChain LLM (chainLlm)         | Writes final article based on research               | Aggregate full research       | Article result                    | ## AI step writing the final article based on the research and initial request                 |
| GPT 5 mini                    | LangChain LLM (lmChatOpenAi)      | AI model for generating research questions          | -                             | New content - generate research questions |                                                                                                |
| GPT 5 chat                   | LangChain LLM (lmChatOpenAi)      | AI model for final article writing                   | -                             | New content - Generate the AI output |                                                                                                |
| Article result               | Set                              | Stores the final article text                         | New content - Generate the AI output | -                                  |                                                                                                |
| Sticky Note                  | Sticky Note                      | Instructional note on Lookio credentials             | -                             | -                                  | Connect your [lookio.app](https://www.lookio.app) credentials & set the ID of the Assistant to query |
| Sticky Note1                 | Sticky Note                      | Overview and usage instructions                       | -                             | -                                  | # Expert AI Article Writer - Knowledge base + Web - Detailed workflow explanation by Guillaume Duvernay |
| Sticky Note2                 | Sticky Note                      | Notes on answering each sub-question with dual sources | -                             | -                                  | ## Answering each sub-question one by one with Lookio for internal knowledge insights & Linkup for web-insights |
| Sticky Note3                 | Sticky Note                      | Notes on AI final writing step                        | -                             | -                                  | ## AI step writing the final article based on the research and initial request                 |
| Sticky Note4                 | Sticky Note                      | Notes on decomposing topic into sub-questions        | -                             | -                                  | ## Breaking down the topic into sub-questions                                                  |
| Sticky Note5                 | Sticky Note                      | Notes on form usage                                   | -                             | -                                  | ## Fill in this form to request a new article                                                  |
| Sticky Note6                 | Sticky Note                      | Notes on Linkup credentials                           | -                             | -                                  | Connect your linkup.so credentials (adding your API key in the header or using "generic credentials") |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node ("New article form"):**  
   - Type: Form Trigger  
   - Configure form with two required fields:  
     - "Article title" (text)  
     - "Article guidelines" (text)  
   - Enable webhook to trigger workflow on submission.

2. **Add a Set node ("Prepare form values"):**  
   - Copy form fields to variables:  
     - `Title` = `{{ $json["Article title"] }}`  
     - `Guidelines` = `{{ $json["Article guidelines"] }}`  
   - Connect output of Form node to this Set node.

3. **Add LangChain LLM node ("New content - generate research questions"):**  
   - Model: GPT-5 mini  
   - Prompt: Incorporate `Title` and `Guidelines` to instruct AI to generate 5-8 non-overlapping research questions in JSON array format.  
   - Configure output parser (next step) to parse JSON questions.  
   - Connect output of Set node to this node.

4. **Add LangChain Output Parser node ("Structured Output Parser"):**  
   - Use JSON schema example matching expected question array format.  
   - Connect AI output of the previous node to this parser node.

5. **Add Split Out node ("Split Out Questions"):**  
   - Field to split out: parsed questions array.  
   - Connect output of parser node to this Split Out node.

6. **Add Split In Batches node ("Loop Over Questions"):**  
   - Use default batch size = 1 to process questions one by one.  
   - Connect output of Split Out node here.

7. **Add HTTP Request node ("Query Lookio Assistant"):**  
   - Method: POST  
   - URL: `https://api.lookio.app/webhook/query`  
   - Headers: `api_key` with your Lookio API Token  
   - Body (JSON):  
     - `query`: current question (`{{$json.question}}`)  
     - `assistant_id`: your Lookio Assistant ID  
     - `query_mode`: "flash"  
   - Connect output of Loop Over Questions to this node.

8. **Add HTTP Request node ("Query Linkup for AI web-search"):**  
   - Method: POST  
   - URL: `https://api.linkup.so/v1/search`  
   - Authentication: HTTP Bearer using stored Linkup API credentials  
   - Body: JSON with parameters:  
     - `q`: prompt requesting 5 specific, meaningful insights related to current question  
     - `depth`: "standard"  
     - `outputType`: "structured"  
     - `structuredOutputSchema`: JSON schema defining array of insights with fields `insight`, `url`, `title`  
     - `includeImages`: false  
   - Connect output of Lookio Assistant node to this node.

9. **Add Set node ("Format question and insights"):**  
   - Assign fields:  
     - `Question`: from Loop Over Questions current item (`{{$json.question}}`)  
     - `Internal knowledge insights`: from Lookio response (`{{$json.Output}}`)  
     - `Web insights`: from Linkup response (`{{$json.insights}}`)  
   - Connect output of Linkup node to this node.

10. **Connect output of Format node back to Loop Over Questions:**  
    - This enables batching and iterative processing.

11. **Add Aggregate node ("Aggregate full research"):**  
    - Aggregate all research outputs into a single JSON array field named `Content to leverage`.  
    - Connect one output of Loop Over Questions to this node.

12. **Add LangChain LLM node ("New content - Generate the AI output"):**  
    - Model: GPT-5 chat latest  
    - Prompt:  
      - Include article title, guidelines, and the aggregated JSON research (`Content to leverage`) as content to leverage.  
      - Instruct AI to write a human-like article with headings and embedded source links.  
    - Connect output of Aggregate node to this node.

13. **Add Set node ("Article result"):**  
    - Store output text from AI generation in field `Article`.  
    - Connect output of AI generation node here.

14. **Configure Credentials:**  
    - OpenAI API credentials with GPT-5 access for LangChain nodes.  
    - Lookio API key and Assistant ID in the HTTP Request node for Lookio.  
    - Linkup API credentials stored as HTTP Bearer and linked to respective HTTP Request node.

15. **Add Sticky Notes as needed for instructions and reminders:**  
    - Include notes about credential setup, usage instructions, and overview of blocks.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                  | Context or Link                                    |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------|---------------------------------------------------|
| This workflow was built by Guillaume Duvernay to create expert AI articles combining internal knowledge and web-based insights.                             | Author credit                                     |
| Connect your [lookio.app](https://www.lookio.app) credentials and set your Assistant ID in the corresponding HTTP Request node.                             | Lookio API setup                                  |
| Connect your [linkup.so](https://linkup.so) credentials by adding your API key in the header or using stored generic credentials for the Linkup HTTP node.   | Linkup API setup                                  |
| The workflow breaks down the article topic into sub-questions, queries both internal and web sources for each, then writes an article grounded in research.    | Workflow high-level logic                          |
| The article writing AI uses GPT-5 chat latest model; research question generation uses GPT-5 mini model for efficiency.                                       | AI model usage                                    |

---

This completes the comprehensive reference documentation for the "Create Dual-Source Expert Articles with Internal Knowledge and Web Research using Lookio, Linkup, and GPT-5" workflow.