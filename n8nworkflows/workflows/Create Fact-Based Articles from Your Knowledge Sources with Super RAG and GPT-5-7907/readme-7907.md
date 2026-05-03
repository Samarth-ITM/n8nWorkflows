Create Fact-Based Articles from Your Knowledge Sources with Super RAG and GPT-5

https://n8nworkflows.xyz/workflows/create-fact-based-articles-from-your-knowledge-sources-with-super-rag-and-gpt-5-7907


# Create Fact-Based Articles from Your Knowledge Sources with Super RAG and GPT-5

### 1. Workflow Overview

This workflow automates the generation of fact-based articles using a hybrid AI and knowledge retrieval approach known as Super RAG (Retrieval-Augmented Generation) combined with GPT-5 language models. It targets content creators who want to produce high-quality, research-backed articles grounded in their own curated knowledge sources (e.g., Notion, Drive) integrated via the Super assistant platform.

The workflow is structured into four main logical blocks:

- **1.1 Input Reception:** Captures user input for new article requests via a form and prepares the data for processing.
- **1.2 Question Decomposition:** Uses GPT-5 mini to generate a set of well-structured, non-overlapping research questions that comprehensively cover the article topic.
- **1.3 Research and Answer Aggregation:** Queries the Super assistant API for answers to each generated question, collects and formats these Q&A pairs as structured research content.
- **1.4 Article Generation:** Uses GPT-5 chat to write a final article strictly based on the aggregated research content and user guidelines, embedding source references as hyperlinks.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block triggers the workflow by receiving user inputs (article title and guidelines) from a web form, then prepares these values for downstream processing.

**Nodes Involved:**  
- New article form  
- Prepare form values

**Node Details:**  

- **New article form**  
  - *Type:* Form Trigger  
  - *Role:* Web form interface to collect article title and guidelines from the user.  
  - *Configuration Highlights:* Two required fields: "Article title" and "Article guidelines". Triggered via webhook.  
  - *Input:* User-submitted form data  
  - *Output:* JSON containing the submitted fields.  
  - *Edge Cases:* Missing required fields block execution; webhook availability required.

- **Prepare form values**  
  - *Type:* Set node  
  - *Role:* Extracts and standardizes form data fields into named variables ("Title" and "Guidelines") for later nodes.  
  - *Key Expressions:*  
    - Title: `={{ $json['Article title'] }}`  
    - Guidelines: `={{ $json['Article guidelines'] }}`  
  - *Input:* Output of form node  
  - *Output:* JSON object with properties "Title" and "Guidelines"  
  - *Failure Points:* Expression errors if input fields missing.

---

#### 2.2 Question Decomposition

**Overview:**  
This block generates a set of 5–8 well-structured research questions to cover the article topic comprehensively, following a guideline to start broad and then get specific.

**Nodes Involved:**  
- New content - generate research questions  
- Structured Output Parser  
- Split Out Questions  
- Loop Over Questions

**Node Details:**  

- **New content - generate research questions**  
  - *Type:* LangChain LLM chain (GPT-5 mini)  
  - *Role:* Generates JSON array of research questions based on the article title and guidelines.  
  - *Prompt Highlights:* Instructs model to create non-overlapping questions that cover broad to advanced topics, outputting a JSON array with question strings.  
  - *Key Expressions:*  
    - Input text includes interpolated title and guidelines:  
      ```
      =Content title:  {{ $json.Title }}

      Article guidelines: {{ $json.Guidelines }}
      ```  
  - *Input:* Prepared form values (Title, Guidelines)  
  - *Output:* JSON array of questions (as a string)  
  - *Edge Cases:* Model may generate malformed JSON; fallback or parser errors possible.

- **Structured Output Parser**  
  - *Type:* LangChain JSON output parser  
  - *Role:* Parses the raw string output from LLM into structured JSON array format.  
  - *Config:* Example schema with array of objects containing "question" string.  
  - *Input:* Raw LLM output  
  - *Output:* JSON array object  
  - *Failure Points:* Parsing errors if LLM output invalid.

- **Split Out Questions**  
  - *Type:* Split Out  
  - *Role:* Converts the array of questions into individual items for batch processing.  
  - *Input:* Parsed JSON array  
  - *Output:* Stream of single question JSON objects  
  - *Edge Cases:* Empty or malformed arrays cause no output.

- **Loop Over Questions**  
  - *Type:* Split In Batches  
  - *Role:* Iterates over individual questions for sequential processing in research phase.  
  - *Input:* Single question items  
  - *Output:* Batches (default size 1) of question items for next block  
  - *Edge Cases:* Empty input halts workflow.

---

#### 2.3 Research and Answer Aggregation

**Overview:**  
For each question generated, this block queries the Super assistant API to retrieve relevant answers grounded in the user’s knowledge sources, formats the Q&A pairs, and aggregates them into a single research content dataset.

**Nodes Involved:**  
- Query Super Assistant  
- Format question and answer  
- Aggregate research content  
- Sticky Note (for credentials reminder)

**Node Details:**  

- **Query Super Assistant**  
  - *Type:* HTTP Request  
  - *Role:* Sends each question to the Super assistant API to get an answer based on connected knowledge bases.  
  - *Configuration Highlights:*  
    - POST to `https://api.super.work/v1/super`  
    - Body includes `question` (from current batch item) and `assistantId` (to be replaced with user’s ID).  
  - *Input:* Question JSON with "question" property  
  - *Output:* API JSON response containing answer text and sources  
  - *Version Requirements:* API token and valid assistant ID required (set outside node)  
  - *Failure Types:* Network errors, auth failures, invalid assistant ID, rate limits.  
  - *Sticky Note:* Reminds user to connect Super.com credentials and replace assistant ID.

- **Format question and answer**  
  - *Type:* Set node  
  - *Role:* Extracts and formats the question and answer text, replacing Super’s markdown-style source references with standard markdown hyperlinks.  
  - *Key Expressions:*  
    - Question: `={{ $('Loop Over Questions').item.json.question }}`  
    - Answer: `={{ $json.answer.replace(/\{\[(\d+)\]\((.*?)\)/g, '([source]($2))') }}`  
  - *Input:* API response from Super assistant  
  - *Output:* JSON with "Question" and cleaned "Answer" strings  
  - *Edge Cases:* Regex may fail if source format changes.

- **Aggregate research content**  
  - *Type:* Aggregate node  
  - *Role:* Merges all individual Q&A pairs into a single field "Content to leverage" for final article generation.  
  - *Input:* Stream of formatted Q&A JSON objects  
  - *Output:* One aggregated JSON object containing all research content  
  - *Edge Cases:* Empty input leads to empty aggregate.

---

#### 2.4 Article Generation

**Overview:**  
This block synthesizes the final article by feeding the aggregated research content, article title, and guidelines into GPT-5 chat, instructing it to write a complete, human-like article including source hyperlinks.

**Nodes Involved:**  
- New content - Generate the AI output  
- GPT 5 chat  
- Article result  
- Sticky Note (describing AI writing step)

**Node Details:**  

- **New content - Generate the AI output**  
  - *Type:* LangChain LLM chain (GPT-5 chat)  
  - *Role:* Composes the article text strictly based on the research content and user guidelines.  
  - *Prompt Highlights:*  
    - Emphasizes using only provided research, including source links as hyperlinks.  
    - Output must be a complete article starting with an H1 title and subheadings.  
  - *Key Expressions:*  
    - Inputs interpolated from the "Prepare form values" node and aggregated research content as a formatted JSON string.  
  - *Input:* Aggregated research content, title, guidelines  
  - *Output:* Raw article text  
  - *Failure Points:* Model hallucination if prompt misunderstood; API limit or auth errors.

- **GPT 5 chat**  
  - *Type:* LangChain OpenAI Chat model node (GPT-5 chat-latest)  
  - *Role:* Executes the final article generation LLM call based on the prompt from previous node.  
  - *Input:* Prompt text from "New content - Generate the AI output" node  
  - *Output:* Generated article text  
  - *Version Specific:* Requires OpenAI credentials with GPT-5 chat access.

- **Article result**  
  - *Type:* Set node  
  - *Role:* Stores the final article text in a variable named "Article" for downstream use or output.  
  - *Key Expression:* `={{ $json.text }}`  
  - *Input:* Output of GPT 5 chat node  
  - *Output:* JSON with "Article" property  
  - *Edge Cases:* Empty or partial output if previous node fails.

---

### 3. Summary Table

| Node Name                      | Node Type                       | Functional Role                         | Input Node(s)                 | Output Node(s)                     | Sticky Note                                                                                 |
|--------------------------------|--------------------------------|---------------------------------------|------------------------------|----------------------------------|---------------------------------------------------------------------------------------------|
| New article form                | Form Trigger                   | Receives article title and guidelines | (Trigger)                    | Prepare form values               | Fill in this form to request a new article                                                  |
| Prepare form values             | Set                           | Standardizes form inputs               | New article form              | New content - generate research questions |                                                                                             |
| New content - generate research questions | LangChain LLM chain (GPT-5 mini) | Generates research questions           | Prepare form values           | Structured Output Parser          | Breaking down the topic into sub-questions                                                 |
| Structured Output Parser        | LangChain Output Parser Structured | Parses raw string to JSON array       | New content - generate research questions | Split Out Questions              |                                                                                             |
| Split Out Questions             | Split Out                     | Splits questions array into individual items | Structured Output Parser      | Loop Over Questions              |                                                                                             |
| Loop Over Questions             | Split In Batches              | Iterates over questions for querying  | Split Out Questions           | Query Super Assistant, Aggregate research content | Answering each sub-question one by one with Super                                          |
| Query Super Assistant           | HTTP Request                  | Gets answers for questions from Super assistant API | Loop Over Questions           | Format question and answer       | Connect your super.com credentials & replace your assistant ID                             |
| Format question and answer      | Set                           | Formats Q&A pairs, replaces source refs | Query Super Assistant         | Loop Over Questions              |                                                                                             |
| Aggregate research content      | Aggregate                     | Merges all Q&A pairs into research content | Loop Over Questions            | New content - Generate the AI output |                                                                                             |
| New content - Generate the AI output | LangChain LLM chain (GPT-5 chat) | Produces final article based on research | Aggregate research content    | GPT 5 chat                      | AI step writing the final article based on the research and initial request                |
| GPT 5 chat                     | LangChain OpenAI Chat Model   | Executes GPT-5 chat model call         | New content - Generate the AI output | New content - Generate the AI output |                                                                                             |
| Article result                 | Set                           | Stores final article text              | New content - Generate the AI output | (End)                          |                                                                                             |
| Sticky Note                    | Sticky Note                   | Reminder for Super assistant credentials | (none)                      | (none)                          | Connect your super.com credentials & replace your assistant ID                             |
| Sticky Note1                   | Sticky Note                   | Overview and usage instructions        | (none)                      | (none)                          | AI Article Writer Based on Your Knowledge Base by Guillaume Duvernay                       |
| Sticky Note2                   | Sticky Note                   | Notes on answering sub-questions       | (none)                      | (none)                          | Answering each sub-question one by one with Super                                          |
| Sticky Note3                   | Sticky Note                   | Notes on AI writing step                | (none)                      | (none)                          | AI step writing the final article based on the research and initial request                |
| Sticky Note4                   | Sticky Note                   | Notes on question breakdown             | (none)                      | (none)                          | Breaking down the topic into sub-questions                                                 |
| Sticky Note5                   | Sticky Note                   | Notes on form usage                     | (none)                      | (none)                          | Fill in this form to request a new article                                                  |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "New article form" node (Form Trigger):**  
   - Set form title to "New article".  
   - Add two required fields:  
     - "Article title" (e.g. placeholder: "10 ways to do Influencer Marketing in 2025")  
     - "Article guidelines" (e.g. placeholder: "Promote xyz and write in British English...")  
   - Save webhook ID for external triggers.

2. **Create "Prepare form values" node (Set):**  
   - Connect from "New article form".  
   - Create two string fields:  
     - Title: `={{ $json['Article title'] }}`  
     - Guidelines: `={{ $json['Article guidelines'] }}`  

3. **Create "New content - generate research questions" node (LangChain chain LLM):**  
   - Connect from "Prepare form values".  
   - Set model to "gpt-5-mini".  
   - Configure input text:  
     ```
     =Content title:  {{ $json.Title }}

     Article guidelines: {{ $json.Guidelines }}
     ```  
   - Use a message prompt instructing to generate 5-8 non-overlapping questions in JSON array format, starting broad then specific, covering the topic fully.  
   - Enable output parser.

4. **Create "Structured Output Parser" node (LangChain output parser structured):**  
   - Connect from "New content - generate research questions".  
   - Provide an example JSON schema for the array of questions (each with "question" string).

5. **Create "Split Out Questions" node (Split Out):**  
   - Connect from "Structured Output Parser".  
   - Set fieldToSplitOut to the array field containing questions.

6. **Create "Loop Over Questions" node (Split In Batches):**  
   - Connect from "Split Out Questions".  
   - Use default batch size = 1 for sequential processing.

7. **Create "Query Super Assistant" node (HTTP Request):**  
   - Connect from "Loop Over Questions".  
   - Set method to POST, URL: `https://api.super.work/v1/super`.  
   - In body parameters, add:  
     - `question`: `={{ $json.question }}`  
     - `assistantId`: replace with your Super assistant ID (string).  
   - Set authentication and API token for Super.com in credentials.  
   - Set content-type to JSON.

8. **Create "Format question and answer" node (Set):**  
   - Connect from "Query Super Assistant".  
   - Create two fields:  
     - Question: `={{ $('Loop Over Questions').item.json.question }}`  
     - Answer: `={{ $json.answer.replace(/\{\[(\d+)\]\((.*?)\)/g, '([source]($2))') }}` to replace source references with markdown links.

9. **Connect "Format question and answer" back to "Loop Over Questions"** to continue iteration.

10. **Create "Aggregate research content" node (Aggregate):**  
    - Connect from "Loop Over Questions" (on main output after formatting).  
    - Set aggregation to "aggregate all item data" into field "Content to leverage".

11. **Create "New content - Generate the AI output" node (LangChain chain LLM):**  
    - Connect from "Aggregate research content".  
    - Use model "gpt-5-chat-latest".  
    - Configure input text with interpolation of:  
      - Article title and guidelines from "Prepare form values" node (first item).  
      - Stringified research content from aggregation.  
    - Provide a detailed prompt instructing the model to write a complete article based solely on the research content, including source links as hyperlinks, starting with H1 title and subheadings.

12. **Create "GPT 5 chat" node (LangChain OpenAI Chat Model):**  
    - Connect from "New content - Generate the AI output".  
    - Model: "gpt-5-chat-latest".  
    - Set credentials for OpenAI with GPT-5 chat access.

13. **Create "Article result" node (Set):**  
    - Connect from "GPT 5 chat".  
    - Create a field "Article" with value `={{ $json.text }}` to store the generated article text.

14. **Add Sticky Notes where appropriate:**  
    - Credential reminders near "Query Super Assistant".  
    - Descriptions for blocks (input form, question generation, research, final writing).  
    - Usage instructions and overview.

15. **Configure Credentials:**  
    - Connect OpenAI credentials with GPT-5 access to LangChain LLM nodes.  
    - Connect Super.com API token and assistant ID for HTTP Request node.

16. **Test the entire flow:**  
    - Trigger via the form with sample inputs.  
    - Verify questions generate correctly.  
    - Confirm Super assistant answers are retrieved.  
    - Check final article generation includes source links and follows guidelines.

---

### 5. General Notes & Resources

| Note Content                                                                                                          | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| AI Article Writer Based on Your Knowledge Base: automates research and writing grounded in your own knowledge sources. | Sticky Note1 content by Guillaume Duvernay                                                       |
| Connect your super.com credentials & replace your assistant ID                                                         | Credential reminder for Super assistant API node                                                 |
| Workflow uses GPT-5 mini for question generation and GPT-5 chat for final article writing                             | Model selection for balance of speed and quality                                                 |
| Workflow requires creating a Super assistant with your knowledge sources connected via Super.com platform             | External setup prerequisite: https://super.com                                                   |
| The article output is formatted to include source links as markdown hyperlinks for transparency and validation        | Included in final AI prompt instructions                                                        |
| The form trigger enables easy external invocation or embedding in web apps                                           | Webhook URL available after deployment                                                           |

---

This document comprehensively describes the "Create Fact-Based Articles from Your Knowledge Sources with Super RAG and GPT-5" workflow, enabling detailed understanding, reproduction, and modification with full context on integration points and potential failure modes.

---

*Disclaimer: The provided content originates exclusively from an automated workflow created with n8n, adhering strictly to current content policies and containing no illegal or protected data. All processed data is legal and public.*