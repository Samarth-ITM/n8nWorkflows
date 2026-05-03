Interactive Structured Prompt Builder with GPT-4.1-mini and Guided Questions

https://n8nworkflows.xyz/workflows/interactive-structured-prompt-builder-with-gpt-4-1-mini-and-guided-questions-11361


# Interactive Structured Prompt Builder with GPT-4.1-mini and Guided Questions

### 1. Workflow Overview

This workflow, titled **Interactive Structured Prompt Builder with GPT-4.1-mini and Guided Questions**, is designed to facilitate the incremental construction of AI prompts through a guided, interactive question-and-answer interface. It leverages GPT-based language models to generate, refine, and structure these prompts dynamically based on user inputs captured through web forms. The main use case is to help users build complex AI prompts step-by-step, ensuring structured, validated output that can be used for further AI processing.

The workflow logic is divided into the following functional blocks:

- **1.1 Input Reception and Initial Processing**  
  Captures initial user inputs through web forms and triggers the AI prompt generation process.

- **1.2 AI-Driven Question Generation and Loop**  
  Uses AI models to generate relevant follow-up questions, iteratively refining the prompt via a controlled loop with batch processing.

- **1.3 Response Processing and Output Structuring**  
  Parses and structures AI responses, fixes output inconsistencies, and prepares the final prompt for submission.

- **1.4 User Interaction and Form Management**  
  Manages multiple forms for base questions, looped relevant questions, and final prompt submission, handling user interaction flow.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Initial Processing

- **Overview:**  
  This block captures the initial user input via a web form and starts the prompt generation chain. It ensures incoming data is merged and split appropriately for AI processing.

- **Nodes Involved:**  
  - Form submission  
  - Base question  
  - ChainLlm AI  
  - Merge  
  - Prompt generator  
  - Send prompt

- **Node Details:**

  - **Form submission**  
    - Type: Form Trigger (webhook)  
    - Role: Receives the initial user input to start the workflow  
    - Configuration: Uses a webhook ID (`5ce3c206-6c8e-4869-8123-27eb5fe6809b`) to listen for form submissions  
    - Input: External HTTP form submission  
    - Output: Data to "Base question" node  
    - Potential Failures: Webhook timeout, missing data  
    - Version: 2.2  

  - **Base question**  
    - Type: Form (webhook)  
    - Role: Presents base questions to the user for initial prompt refinement  
    - Configuration: Uses webhook ID (`cdf6ec24-43fe-4a6b-908a-9d17d3e0feec`) for form interaction  
    - Input: From "Form submission" node  
    - Output: Data to "ChainLlm AI" and "Merge" and "Split question"  
    - Potential Failures: User input errors, form validation issues  
    - Version: 1  

  - **ChainLlm AI**  
    - Type: LangChain LLM Chain (OpenAI GPT model)  
    - Role: Processes base questions and generates AI responses for prompt building  
    - Configuration: Uses a GPT model (likely GPT-4.1-mini based on workflow title)  
    - Input: From "Base question"  
    - Output: To "Split question" node  
    - Potential Failures: API auth errors, rate limits, timeout  
    - Version: 1.6  

  - **Merge**  
    - Type: Merge node  
    - Role: Combines data streams from "Base question" and from the loop question batch processing (see later blocks)  
    - Input: From "Base question" and "Loop question" nodes  
    - Output: To "Prompt generator"  
    - Potential Failures: Data mismatch, empty inputs  
    - Version: 3.1  

  - **Prompt generator**  
    - Type: LangChain LLM Chain  
    - Role: Generates the next prompt iteration based on merged inputs  
    - Configuration: Uses GPT model, likely configured with prompt templates  
    - Input: From "Merge" and "OpenAI1" nodes  
    - Output: To "Send prompt"  
    - ExecuteOnce: true (runs once per trigger)  
    - Potential Failures: OpenAI API errors, prompt formatting issues  
    - Version: 1.6  

  - **Send prompt**  
    - Type: Form (webhook)  
    - Role: Presents the generated prompt back to the user for final input or confirmation  
    - Configuration: Uses webhook ID (`9a0af1f5-3257-486c-ad6e-068a7c0f93ab`)  
    - Input: From "Prompt generator"  
    - Output: To no further node (end of chain here)  
    - Potential Failures: Form submission errors  
    - Version: 1  

#### 2.2 AI-Driven Question Generation and Loop

- **Overview:**  
  This block iteratively generates relevant follow-up questions to refine the prompt further. It splits questions into batches, presents them to users via forms, collects answers, and merges responses for further processing.

- **Nodes Involved:**  
  - Split question  
  - Loop question  
  - Relevant question

- **Node Details:**

  - **Split question**  
    - Type: Split Out  
    - Role: Splits AI-generated questions into individual elements to process in batches  
    - Input: From "ChainLlm AI"  
    - Output: To "Loop question"  
    - Potential Failures: Empty data, splitting errors  
    - Version: 1  

  - **Loop question**  
    - Type: Split In Batches  
    - Role: Controls batch processing of questions to manage user interaction load  
    - Configuration: Executes multiple times, not once  
    - Input: From "Split question"  
    - Output: To "Relevant question" and "Merge"  
    - Potential Failures: Batch size misconfiguration, infinite loops  
    - Version: 3  

  - **Relevant question**  
    - Type: Form (webhook)  
    - Role: Presents relevant follow-up questions to the user and captures answers  
    - Configuration: Webhook ID (`8d220b4e-1c9c-4f52-8d1f-0e5f396f7745`)  
    - Input: From "Loop question"  
    - Output: To "Loop question" (feedback loop)  
    - Potential Failures: User input errors, form timeout  
    - Version: 1  

#### 2.3 Response Processing and Output Structuring

- **Overview:**  
  After collecting user answers and AI responses, this block parses, validates, and fixes the output to ensure the prompt is well-structured and consistent.

- **Nodes Involved:**  
  - OpenAI  
  - OpenAI1  
  - Output Parser  
  - Fixing Output  
  - Structured Output

- **Node Details:**

  - **OpenAI**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Provides AI language model processing for intermediate steps  
    - Input: Connected to "ChainLlm AI"  
    - Output: To "ChainLlm AI"  
    - Configuration: Uses GPT-4.1-mini or similar  
    - Potential Failures: API auth, rate limits, malformed requests  
    - Version: 1.2  

  - **OpenAI1**  
    - Type: LangChain OpenAI Chat Model node  
    - Role: Used in prompt generation and output fixing  
    - Input: Connects to "Prompt generator" and "Fixing Output" nodes  
    - Output: To "Prompt generator" and "Fixing Output"  
    - Potential Failures: Same as OpenAI node  
    - Version: 1.2  

  - **Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Role: Parses AI responses into structured JSON or defined format for further use  
    - Input: From "OpenAI1"  
    - Output: To "Fixing Output"  
    - Potential Failures: Parsing errors, unexpected output format  
    - Version: 1.2  

  - **Fixing Output**  
    - Type: LangChain Autofixing Output Parser  
    - Role: Attempts to automatically fix parsing errors or inconsistencies in AI output  
    - Input: From "Output Parser" and "OpenAI1"  
    - Output: To "Prompt generator" for reprocessing if needed  
    - Potential Failures: Endless loops if fixing fails, API errors  
    - Version: 1  

  - **Structured Output**  
    - Type: LangChain Structured Output Parser  
    - Role: Final parsing to produce clean structured output fed back to "ChainLlm AI"  
    - Input: From "OpenAI"  
    - Output: To "ChainLlm AI"  
    - Potential Failures: Parsing errors  
    - Version: 1.2  

#### 2.4 User Interaction and Form Management

- **Overview:**  
  This block manages user-facing forms, coordinating between initial questions, iterative relevant questions, and final prompt presentation.

- **Nodes Involved:**  
  - Form submission  
  - Base question  
  - Relevant question  
  - Send prompt

- **Node Details:**

  - **Form submission** (already documented in 2.1)  
  - **Base question** (already documented in 2.1)  
  - **Relevant question** (already documented in 2.2)  
  - **Send prompt** (already documented in 2.1)  

---

### 3. Summary Table

| Node Name         | Node Type                                  | Functional Role                                | Input Node(s)                     | Output Node(s)                         | Sticky Note                      |
|-------------------|--------------------------------------------|-----------------------------------------------|----------------------------------|---------------------------------------|---------------------------------|
| Form submission   | Form Trigger (webhook)                      | Captures initial user input                    | External HTTP                    | Base question                         |                                 |
| Base question     | Form (webhook)                             | Presents initial questions to user             | Form submission                 | ChainLlm AI, Merge, Split question    |                                 |
| ChainLlm AI       | LangChain LLM Chain                        | Processes base questions for AI prompt         | Base question, Structured Output | Split question                        |                                 |
| Merge             | Merge node                                | Merges data streams from base and loop question | Base question, Loop question     | Prompt generator                      |                                 |
| Prompt generator  | LangChain LLM Chain                        | Generates prompt iterations                      | Merge, OpenAI1                  | Send prompt                          |                                 |
| Send prompt       | Form (webhook)                            | Presents final prompt to the user                | Prompt generator                | None                                |                                 |
| Split question    | Split Out                                | Splits AI-generated questions into elements      | ChainLlm AI                    | Loop question                       |                                 |
| Loop question     | Split In Batches                         | Iteratively processes questions in batches       | Split question                 | Merge, Relevant question             |                                 |
| Relevant question | Form (webhook)                           | Presents relevant questions and captures answers | Loop question                  | Loop question                      |                                 |
| OpenAI            | LangChain OpenAI Chat Model              | AI processing for intermediate steps             | ChainLlm AI                    | ChainLlm AI                        |                                 |
| OpenAI1           | LangChain OpenAI Chat Model              | Used in prompt generation and output fixing      | Prompt generator, Fixing Output | Prompt generator, Fixing Output     |                                 |
| Output Parser     | LangChain Structured Output Parser       | Parses AI responses into structured format         | OpenAI1                       | Fixing Output                     |                                 |
| Fixing Output     | LangChain Autofixing Output Parser        | Auto-fixes parsing errors in AI output             | Output Parser, OpenAI1          | Prompt generator                   |                                 |
| Structured Output | LangChain Structured Output Parser       | Produces final structured output                   | OpenAI                        | ChainLlm AI                       |                                 |
| Sticky Note1      | Sticky Note                              | (Empty content)                                   | None                          | None                              |                                 |
| Sticky Note3      | Sticky Note                              | (Empty content)                                   | None                          | None                              |                                 |
| Sticky Note4      | Sticky Note                              | (Empty content)                                   | None                          | None                              |                                 |
| Sticky Note5      | Sticky Note                              | (Empty content)                                   | None                          | None                              |                                 |
| Sticky Note6      | Sticky Note                              | (Empty content)                                   | None                          | None                              |                                 |
| Sticky Note7      | Sticky Note                              | (Empty content)                                   | None                          | None                              |                                 |
| Sticky Note       | Sticky Note                              | (Empty content)                                   | None                          | None                              |                                 |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node "Form submission":**  
   - Type: Form Trigger  
   - Configure webhook ID to listen for initial form submissions (`5ce3c206-6c8e-4869-8123-27eb5fe6809b`).  
   - This node starts the workflow when a user submits the base prompt input.

2. **Create Form node "Base question":**  
   - Type: Form  
   - Configure webhook ID (`cdf6ec24-43fe-4a6b-908a-9d17d3e0feec`).  
   - Connect input from "Form submission".  
   - This node presents the base questions for initial prompt building.

3. **Create LangChain LLM Chain node "ChainLlm AI":**  
   - Type: LangChain Chain LLM  
   - Configure with GPT-4.1-mini or similar model credentials.  
   - Connect input from "Base question" (main output).  
   - This node generates AI responses based on base questions.

4. **Create "Merge" node:**  
   - Type: Merge  
   - Connect inputs from "Base question" (secondary output) and "Loop question" (to be created later).  
   - This node merges base and loop question responses.

5. **Create LangChain LLM Chain node "Prompt generator":**  
   - Type: LangChain Chain LLM  
   - Configure with GPT model credentials.  
   - Connect input from "Merge".  
   - Set execute once to true.  
   - This node generates the refined prompt after merging inputs.

6. **Create Form node "Send prompt":**  
   - Type: Form  
   - Configure webhook ID (`9a0af1f5-3257-486c-ad6e-068a7c0f93ab`).  
   - Connect input from "Prompt generator".  
   - This node sends the generated prompt back to the user.

7. **Create "Split question" node:**  
   - Type: Split Out  
   - Connect input from "ChainLlm AI".  
   - This node splits generated questions for batch iteration.

8. **Create "Loop question" node:**  
   - Type: Split In Batches  
   - Connect input from "Split question".  
   - Configure batch size as per desired user interaction chunk size.  
   - This node manages iterative question batches.

9. **Create Form node "Relevant question":**  
   - Type: Form  
   - Configure webhook ID (`8d220b4e-1c9c-4f52-8d1f-0e5f396f7745`).  
   - Connect input from "Loop question".  
   - Connect output back to "Loop question" to create a loop.  
   - This node presents relevant questions iteratively to the user.

10. **Create LangChain OpenAI Chat Model node "OpenAI":**  
    - Type: LangChain OpenAI Chat  
    - Configure with OpenAI API credentials for GPT-4.1-mini or similar.  
    - Connect input from "ChainLlm AI".  
    - Connect output back to "ChainLlm AI".

11. **Create LangChain OpenAI Chat Model node "OpenAI1":**  
    - Type: LangChain OpenAI Chat  
    - Configure with OpenAI API credentials.  
    - Connect inputs from "Prompt generator" and "Fixing Output".  
    - Connect outputs to "Prompt generator" and "Fixing Output".

12. **Create LangChain Structured Output Parser node "Output Parser":**  
    - Type: Structured Output Parser  
    - Connect input from "OpenAI1".  
    - Connect output to "Fixing Output".

13. **Create LangChain Autofixing Output Parser node "Fixing Output":**  
    - Type: Autofixing Output Parser  
    - Connect inputs from "Output Parser" and "OpenAI1".  
    - Connect output to "Prompt generator".

14. **Create LangChain Structured Output Parser node "Structured Output":**  
    - Type: Structured Output Parser  
    - Connect input from "OpenAI".  
    - Connect output to "ChainLlm AI".

15. **Connect "Form submission" to "Base question".**  
16. **Connect "Base question" to "ChainLlm AI", "Merge", and "Split question".**  
17. **Connect "Split question" to "Loop question".**  
18. **Connect "Loop question" to "Merge" and "Relevant question".**  
19. **Connect "Relevant question" back to "Loop question".**  
20. **Connect "Merge" to "Prompt generator".**  
21. **Connect "Prompt generator" to "Send prompt".**  
22. **Set up all required credentials:**  
    - OpenAI API key with access to GPT-4.1-mini or equivalent.  
    - Ensure webhook URLs are accessible for all form nodes.

23. **Test the workflow end-to-end:**  
    - Submit initial data via "Form submission".  
    - Verify the iterative question loop and prompt refinement.  
    - Confirm final prompt output in "Send prompt" form.

---

### 5. General Notes & Resources

| Note Content                                                                                          | Context or Link                                  |
|-----------------------------------------------------------------------------------------------------|-------------------------------------------------|
| The workflow uses the GPT-4.1-mini model (or equivalent) for language model nodes, requiring valid OpenAI API credentials. | OpenAI API documentation: https://platform.openai.com/docs/api-reference |
| The iterative question loop uses batch splitting to ensure manageable user interaction chunks.      | N8N SplitInBatches node documentation            |
| Webhook nodes require publicly accessible endpoints; configure n8n with proper tunneling if testing locally. | n8n Webhook docs: https://docs.n8n.io/nodes/n8n-nodes-base.webhook/          |
| Autofixing output parser helps mitigate malformed AI responses by retrying with corrections.         | LangChain parser docs: https://python.langchain.com/en/latest/modules/output_parsers.html |
| The workflow is modular and can be adapted to other AI prompt building scenarios by modifying prompt templates and forms. |                                                 |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, an integration and automation tool. This processing strictly complies with applicable content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.