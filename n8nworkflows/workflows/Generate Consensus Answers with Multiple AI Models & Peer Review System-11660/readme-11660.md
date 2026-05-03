Generate Consensus Answers with Multiple AI Models & Peer Review System

https://n8nworkflows.xyz/workflows/generate-consensus-answers-with-multiple-ai-models---peer-review-system-11660


# Generate Consensus Answers with Multiple AI Models & Peer Review System

### 1. Workflow Overview

This workflow, titled **"Generate Consensus Answers with Multiple AI Models & Peer Review System"**, orchestrates a multi-model AI council approach to generate and refine answers to user questions. It is designed to improve answer robustness and quality by leveraging diverse large language models (LLMs) to independently respond to queries, then critically peer-review each other's answers before producing a final consensus answer.

**Target Use Cases:**  
- Researchers and developers seeking reliable, multi-perspective AI-generated answers  
- Teams evaluating and comparing different LLMs side-by-side  
- Applications requiring critical assessment and synthesis of AI outputs for higher-quality results  

**Logical Blocks:**

- **1.1 Input Reception**: Captures user questions via chat interface.  
- **1.2 Individual Model Response Generation**: Four distinct LLM nodes independently generate answers.  
- **1.3 Aggregation of Answers**: Merges and aggregates all model answers into a unified dataset.  
- **1.4 Peer Review Process**: Four peer review agents each critically assess all submitted answers anonymously.  
- **1.5 Aggregation of Reviews**: Merges and aggregates the peer reviews.  
- **1.6 Final Synthesis**: A final AI agent analyzes all reviews and produces the consensus final answer.  
- **1.7 Documentation and Guidance**: Sticky notes provide instructions, context, and setup guidance throughout the workflow.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Receives the user’s question through an n8n Chat Trigger node, serving as the workflow’s entry point.

- **Nodes Involved:**  
  - Chat Trigger

- **Node Details:**  
  - **Node:** Chat Trigger  
    - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
    - Role: Webhook-based node to receive chat input from user interface  
    - Config: Default options, webhook ID defined for external chat UI integration  
    - Inputs: External user query via webhook  
    - Outputs: Routes question text downstream to answer agents  
    - Edge Cases: Potential webhook failures, input validation errors, or malformed questions  

---

#### 1.2 Individual Model Response Generation

- **Overview:**  
Four language model nodes (Gemini, Llama, Gemma, Mistral) independently generate answers to the input question. Each model runs sequentially, so total duration grows linearly with models.

- **Nodes Involved:**  
  - Gemini Model  
  - Llama Model  
  - Gemma Model  
  - Mistral Model  
  - Answer Agent #1  
  - Answer Agent #2  
  - Answer Agent #3  
  - Answer Agent #4

- **Node Details:**

  - **Gemini Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - Role: AI model node using Google Gemini 2.0 Flash 001 via OpenRouter API  
    - Config: Model set to `"google/gemini-2.0-flash-001"`, connected to OpenRouter credentials  
    - Inputs: Receives prompt from Answer Agent #1  
    - Outputs: Model-generated answer text  
    - Edge Cases: API key expiration, rate limits, model unavailability  

  - **Llama Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - Role: AI model node using Meta Llama 3.2 1b instruct via OpenRouter  
    - Config: Model set to `"meta-llama/llama-3.2-1b-instruct"`  
    - Inputs: Receives prompt from Answer Agent #2  
    - Outputs: Model answer text  
    - Edge Cases: Same as Gemini model  

  - **Gemma Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - Role: AI model node using Google Gemma 3n e4b it via OpenRouter  
    - Inputs: From Answer Agent #3  
    - Outputs: Model answer text  
    - Edge Cases: As above  

  - **Mistral Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - Role: AI model node using MistralAI Mistral Nemo via OpenRouter  
    - Inputs: From Answer Agent #4  
    - Outputs: Model answer text  
    - Edge Cases: As above  

  - **Answer Agents #1 to #4**  
    - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
    - Role: Chain nodes responsible for generating prompts and chaining to respective LLM nodes  
    - Config: Default batching enabled, each connected to specific model node  
    - Inputs: Chat Trigger input question  
    - Outputs: To “Merge Answers” node  
    - Edge Cases: Expression errors in prompt construction, batch execution failures  

---

#### 1.3 Aggregation of Answers

- **Overview:**  
Merges the independent answers from the four answer agents into a single combined dataset.

- **Nodes Involved:**  
  - Merge Answers  
  - Aggregate Answers

- **Node Details:**

  - **Merge Answers**  
    - Type: `n8n-nodes-base.merge`  
    - Role: Merges four inputs (answers) into one data stream for aggregation  
    - Config: Number of inputs set to 4  
    - Inputs: Main outputs from Answer Agents #1 to #4  
    - Outputs: Consolidated answer array to Aggregate Answers  
    - Edge Cases: Missing inputs if any answer agent fails  

  - **Aggregate Answers**  
    - Type: `n8n-nodes-base.aggregate`  
    - Role: Aggregates all merged answers into a single array under `allAnswers` field  
    - Config: Aggregate all item data, destination field `allAnswers`  
    - Inputs: From Merge Answers  
    - Outputs: To all four Review Agents  
    - Edge Cases: Empty or incomplete aggregation if input is missing  

---

#### 1.4 Peer Review Process

- **Overview:**  
Four peer review agents assess all combined answers independently. Each produces a critique listing pros, cons, and overall assessment without knowing answer authorship.

- **Nodes Involved:**  
  - Review Agent #1  
  - Review Agent #2  
  - Review Agent #3  
  - Review Agent #4

- **Node Details:**

  - **Review Agents #1 to #4**  
    - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
    - Role: Peer review chains that analyze all answers and provide structured feedback  
    - Config: Prompt instructs agents to list pros, cons, and overall assessment referencing all answers  
    - Inputs: Aggregated answers (`allAnswers`) from Aggregate Answers node  
    - Outputs: To Merge Reviews node  
    - Edge Cases: Prompt expression errors, incomplete data, API errors  

---

#### 1.5 Aggregation of Reviews

- **Overview:**  
Merges and aggregates all peer review outputs into a single dataset for final analysis.

- **Nodes Involved:**  
  - Merge Reviews  
  - Aggregate Reviews

- **Node Details:**

  - **Merge Reviews**  
    - Type: `n8n-nodes-base.merge`  
    - Role: Merges four peer review inputs into one stream  
    - Config: Number of inputs = 4  
    - Inputs: Outputs from each Review Agent  
    - Outputs: To Aggregate Reviews  
    - Edge Cases: Missing review inputs if any agent fails  

  - **Aggregate Reviews**  
    - Type: `n8n-nodes-base.aggregate`  
    - Role: Aggregates merged reviews into an array `allReviews`  
    - Inputs: From Merge Reviews  
    - Outputs: To Final Analysis Agent  
    - Edge Cases: Aggregation failure on missing data  

---

#### 1.6 Final Synthesis

- **Overview:**  
A final AI agent synthesizes all peer reviews and original answers to produce a concise, consensus-based final answer to the user’s question.

- **Nodes Involved:**  
  - Deepseek R1 model  
  - Final Analysis Agent

- **Node Details:**

  - **Deepseek R1 model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
    - Role: Provides the language model backend for final synthesis  
    - Config: Model set to `"deepseek/deepseek-r1"`, using OpenRouter credentials  
    - Inputs: Prompt from Final Analysis Agent  
    - Outputs: Final synthesized answer  
    - Edge Cases: API errors, model unavailability  

  - **Final Analysis Agent**  
    - Type: `@n8n/n8n-nodes-langchain.agent`  
    - Role: Constructs a prompt with the original question, all answers, and all reviews; instructs the model to produce the best, shortest final answer  
    - Config: Prompt uses expressions to dynamically insert question, answers, and reviews  
    - Inputs: Aggregated reviews from Aggregate Reviews; original question and answers referenced via expressions  
    - Outputs: Final answer output (no further downstream nodes)  
    - Edge Cases: Expression evaluation errors, prompt truncation, API issues  

---

#### 1.7 Documentation and Guidance

- **Overview:**  
Sticky notes provide workflow explanations, usage instructions, and setup guidance for users.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note3  
  - Sticky Note4  
  - Sticky Note5

- **Node Details:**

  - Sticky Note  
    - Content: "## Ask Question" with instructions to enter the question as for a normal LLM

  - Sticky Note1  
    - Content: "## Combine Answers" describing merging of anonymous answers

  - Sticky Note2  
    - Content: "## Individual Model Answers" explaining that each model answers independently

  - Sticky Note3  
    - Content: "## Peer Reviews" describing peer review methodology

  - Sticky Note4  
    - Content: "## Analysis and Final Answer" explaining the final AI synthesis step and customization options

  - Sticky Note5  
    - Content: Detailed workflow overview, inspiration, audience, and setup instructions including OpenRouter credential creation and model swapping tips. Also contains a link to Andrej Karpathy’s LLM Council GitHub for context:  
      https://github.com/karpathy/llm-council

---

### 3. Summary Table

| Node Name          | Node Type                                | Functional Role                         | Input Node(s)                      | Output Node(s)                     | Sticky Note                                      |
|--------------------|----------------------------------------|---------------------------------------|----------------------------------|----------------------------------|-------------------------------------------------|
| Chat Trigger       | @n8n/n8n-nodes-langchain.chatTrigger  | Entry point, receives user question   | (external webhook)               | Answer Agent #1, #2, #3, #4      |                                                 |
| Gemini Model       | lmChatOpenRouter                       | Gemini model generates answer         | Answer Agent #1                  | Answer Agent #1, Review Agent #1 |                                                 |
| Llama Model        | lmChatOpenRouter                       | Llama model generates answer          | Answer Agent #2                  | Answer Agent #2, Review Agent #2 |                                                 |
| Gemma Model        | lmChatOpenRouter                       | Gemma model generates answer          | Answer Agent #3                  | Answer Agent #3, Review Agent #3 |                                                 |
| Mistral Model      | lmChatOpenRouter                       | Mistral model generates answer        | Answer Agent #4                  | Answer Agent #4, Review Agent #4 |                                                 |
| Answer Agent #1    | chainLlm                              | Prepares prompt and chains Gemini     | Chat Trigger                    | Merge Answers                   |                                                 |
| Answer Agent #2    | chainLlm                              | Prepares prompt and chains Llama      | Chat Trigger                    | Merge Answers                   |                                                 |
| Answer Agent #3    | chainLlm                              | Prepares prompt and chains Gemma      | Chat Trigger                    | Merge Answers                   |                                                 |
| Answer Agent #4    | chainLlm                              | Prepares prompt and chains Mistral    | Chat Trigger                    | Merge Answers                   |                                                 |
| Merge Answers      | merge                                | Combines answers from all answer agents| Answer Agents #1-#4             | Aggregate Answers               | Sticky Note1: "## Combine Answers"               |
| Aggregate Answers  | aggregate                            | Aggregates all merged answers          | Merge Answers                  | Review Agents #1-#4             | Sticky Note1: "## Combine Answers"               |
| Review Agent #1    | chainLlm                             | Peer reviews all answers (agent 1)     | Aggregate Answers              | Merge Reviews                  | Sticky Note3: "## Peer Reviews"                   |
| Review Agent #2    | chainLlm                             | Peer reviews all answers (agent 2)     | Aggregate Answers              | Merge Reviews                  | Sticky Note3: "## Peer Reviews"                   |
| Review Agent #3    | chainLlm                             | Peer reviews all answers (agent 3)     | Aggregate Answers              | Merge Reviews                  | Sticky Note3: "## Peer Reviews"                   |
| Review Agent #4    | chainLlm                             | Peer reviews all answers (agent 4)     | Aggregate Answers              | Merge Reviews                  | Sticky Note3: "## Peer Reviews"                   |
| Merge Reviews      | merge                               | Combines peer reviews                   | Review Agents #1-#4            | Aggregate Reviews             | Sticky Note3: "## Peer Reviews"                   |
| Aggregate Reviews  | aggregate                           | Aggregates merged reviews               | Merge Reviews                 | Final Analysis Agent          | Sticky Note4: "## Analysis and Final Answer"     |
| Deepseek R1 model  | lmChatOpenRouter                    | Final model for consensus answer       | Final Analysis Agent           | Final Analysis Agent           | Sticky Note4: "## Analysis and Final Answer"     |
| Final Analysis Agent| agent                              | Synthesizes final answer from reviews  | Aggregate Reviews             | (end)                         | Sticky Note4: "## Analysis and Final Answer"     |
| Sticky Note        | stickyNote                         | Instruction: Ask Question               | None                         | None                         | "## Ask Question"                                 |
| Sticky Note1       | stickyNote                         | Instruction: Combine Answers            | None                         | None                         | "## Combine Answers"                              |
| Sticky Note2       | stickyNote                         | Instruction: Individual Model Answers   | None                         | None                         | "## Individual Model Answers"                     |
| Sticky Note3       | stickyNote                         | Instruction: Peer Reviews                | None                         | None                         | "## Peer Reviews"                                 |
| Sticky Note4       | stickyNote                         | Instruction: Analysis and Final Answer  | None                         | None                         | "## Analysis and Final Answer"                    |
| Sticky Note5       | stickyNote                         | Workflow overview and setup instructions| None                         | None                         | Includes link to https://github.com/karpathy/llm-council |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Chat Trigger Node:**  
   - Type: `@n8n/n8n-nodes-langchain.chatTrigger`  
   - Purpose: Receive user questions via webhook/chat interface  
   - Configuration: Default options, assign webhook ID or let n8n auto-generate  
   - Connect its main output to Answer Agent #1, #2, #3, and #4 nodes  

2. **Create Four Answer Agents (Chain LLM nodes):**  
   - Type: `@n8n/n8n-nodes-langchain.chainLlm`  
   - Name them Answer Agent #1 to #4  
   - Enable default batching  
   - Connect each to a respective model node (see next step)  
   - Connect input from Chat Trigger main output  

3. **Create Four Model Nodes (lmChatOpenRouter):**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenRouter`  
   - Assign models as follows:  
     - Gemini Model: `"google/gemini-2.0-flash-001"`  
     - Llama Model: `"meta-llama/llama-3.2-1b-instruct"`  
     - Gemma Model: `"google/gemma-3n-e4b-it"`  
     - Mistral Model: `"mistralai/mistral-nemo"`  
   - Configure each with OpenRouter credentials  
   - Connect each model output to their corresponding Answer Agent node's AI language model input  

4. **Connect Answer Agents to Merge Node:**  
   - Create a `merge` node named "Merge Answers"  
   - Set inputs to 4  
   - Connect outputs of Answer Agent #1 to #4 to Merge Answers, each as one input  

5. **Create Aggregate Answers Node:**  
   - Type: `aggregate`  
   - Configure to aggregate all incoming items into a field named `allAnswers`  
   - Connect output of Merge Answers to Aggregate Answers  

6. **Create Four Review Agents (chainLlm):**  
   - Type: `chainLlm`  
   - Name Review Agent #1 to #4  
   - Each has a prompt:  
     ```
     You are a critical peer reviewer. Review all the answers provided below and provide:
     1. Pros of each answer
     2. Cons of each answer
     3. Overall assessment

     Original question: {{ $('Chat Trigger').item.json.chatInput }}

     Answers to review:
     1) {{ $json.allAnswers[0].text }}
     2) {{ $json.allAnswers[1].text }}
     3) {{ $json.allAnswers[2].text }}
     4) {{ $json.allAnswers[3].text }}
     ```
   - Input: Connect from Aggregate Answers output  
   - Output: Connect to Merge Reviews node  

7. **Create Merge Reviews Node:**  
   - Type: `merge`  
   - Configure for 4 inputs  
   - Connect outputs from Review Agents #1 to #4 to Merge Reviews  

8. **Create Aggregate Reviews Node:**  
   - Type: `aggregate`  
   - Aggregate all merged review items into field `allReviews`  
   - Connect output of Merge Reviews to Aggregate Reviews  

9. **Create Deepseek R1 Model Node:**  
   - Type: `lmChatOpenRouter`  
   - Model: `"deepseek/deepseek-r1"`  
   - Connect OpenRouter credentials  
   - Connect output to Final Analysis Agent node  

10. **Create Final Analysis Agent Node:**  
    - Type: `agent`  
    - Configure prompt with expression embedding:  
      ```
      You are the final arbitrator for the AI Council. Based on all the peer reviews below, synthesize a comprehensive final answer that:

      Provides the shortest and best possible answer to the original question

      Original question: {{ $('Chat Trigger').item.json.chatInput }}

      Original answers:
      1) {{ $json.allAnswers[0].text }}
      2) {{ $json.allAnswers[1].text }}
      3) {{ $json.allAnswers[2].text }}
      4) {{ $json.allAnswers[3].text }}

      Peer Reviews:
      1) {{ $json.allReviews[0].text }}
      2) {{ $json.allReviews[1].text }}
      3) {{ $json.allReviews[2].text }}
      4) {{ $json.allReviews[3].text }}
      ```
    - Input: Connect from Aggregate Reviews output  
    - Output: Final answer (workflow endpoint)  

11. **Add Sticky Notes for Guidance:**  
    - Create sticky notes with the provided content for instructions, overview, and setup guidance  
    - Optionally include the link to Andrej Karpathy’s LLM Council GitHub:  
      https://github.com/karpathy/llm-council  

12. **Credential Setup:**  
    - In n8n Settings → Credentials, add OpenRouter account with API key  
    - Assign this credential to all `lmChatOpenRouter` nodes  

13. **Activate Workflow:**  
    - Save and activate the workflow  
    - Use the Chat Trigger webhook URL to send questions and receive final consensus answers  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | Context or Link                                                      |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------------|
| Workflow inspired by Andrej Karpathy's LLM Council, a method for multi-model consensus through peer review.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | https://github.com/karpathy/llm-council                            |
| OpenRouter is used as the API gateway to multiple LLMs, enabling flexible model selection and simplified credential management.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | https://openrouter.ai/                                              |
| The workflow includes detailed setup instructions in the Sticky Note5 node, including credential creation and model swapping tips.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          | See Sticky Note5 content in workflow                               |
| The final agent is configured to produce the shortest best answer but can be customized to include reasoning, pros/cons, or other synthesis styles.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Customizable prompt in Final Analysis Agent node                   |
| Execution order is sequential for model responses; adding more models increases processing time linearly. Consider parallel processing enhancements if needed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              | Performance consideration                                          |
| All data handling respects legal and ethical guidelines; no illegal or offensive content is processed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | Disclaimer in initial prompt                                       |

---

**Disclaimer:** The text provided comes exclusively from an automated workflow created with n8n, a tool for integration and automation. This processing strictly complies with current content policies and contains no illegal, offensive, or protected elements. All data handled is legal and public.