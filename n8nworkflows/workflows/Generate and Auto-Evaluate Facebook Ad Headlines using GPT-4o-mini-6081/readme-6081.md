Generate and Auto-Evaluate Facebook Ad Headlines using GPT-4o-mini

https://n8nworkflows.xyz/workflows/generate-and-auto-evaluate-facebook-ad-headlines-using-gpt-4o-mini-6081


# Generate and Auto-Evaluate Facebook Ad Headlines using GPT-4o-mini

### 1. Workflow Overview

This n8n workflow automates the generation and quality evaluation of Facebook ad headlines using GPT-4o-mini via OpenAI. Its primary use case is to take a marketer’s product description through a streamlined process that produces creative ad headlines, objectively scores them on dynamically generated criteria, and decides whether iterations are necessary to improve quality before final output. The workflow is designed for marketing teams and AI practitioners seeking automated, repeatable, and measurable ad copy generation without manual copywriting effort.

The workflow is logically divided into five main blocks:

- **1.1 Input Reception and Prompt Preparation**  
  Captures user input via a web form and builds a consistent prompt for the AI headline generation.

- **1.2 Headline Generation**  
  Uses a language model agent to generate a Facebook ad headline based on the prepared prompt.

- **1.3 Evaluation Criteria Creation**  
  Dynamically creates five scoring parameters (scales 1-10) to objectively assess the headline.

- **1.4 Headline Scoring and Evaluation**  
  Scores the generated headline using the criteria, producing numeric scores and a qualitative bottom line.

- **1.5 Iteration Decision and Output Delivery**  
  Decides whether the headline needs reworking based on evaluation and either finishes by sending the headline via email or loops back for refinement.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception and Prompt Preparation

**Overview:**  
This block receives a product description from a marketer through a web form and constructs a standardized prompt text to be used by the headline generation AI agent.

**Nodes Involved:**  
- FormTrigger_CopywritingBrief  
- Set_PromptForHeadline

**Node Details:**

- **FormTrigger_CopywritingBrief**  
  - Type: Form Trigger (webhook-based)  
  - Role: Captures input with a single required field: "What is your product about?"  
  - Config: Simple form with one required text field  
  - Inputs: External webhook trigger (form submission)  
  - Outputs: JSON containing user’s product description  
  - Failure modes: Missing required field, webhook failure, network errors

- **Set_PromptForHeadline**  
  - Type: Set node  
  - Role: Prepares a prompt string "Write a facebook ad headline for this product:"  
  - Config: Sets static text as “Prompt For Agent” variable  
  - Inputs: From form trigger node  
  - Outputs: JSON with prompt string to feed into AI agent  
  - Failure modes: Expression errors (unlikely since static string), data mapping errors

---

#### 2.2 Headline Generation

**Overview:**  
Generates a Facebook ad headline by feeding the prepared prompt and user input into a GPT-4o-mini powered agent.

**Nodes Involved:**  
- LLM_HeadlineWriterModel  
- Agent_HeadlineWriter

**Node Details:**

- **LLM_HeadlineWriterModel**  
  - Type: Langchain OpenAI Chat LLM node  
  - Role: Provides GPT-4o-mini model capability to agents  
  - Config: Uses n8n OpenAI API credential; model set to "gpt-4o-mini"  
  - Inputs: None directly (called as AI model by Agent node)  
  - Outputs: Model response to Agent node  
  - Failure modes: API auth errors, rate limits, timeouts

- **Agent_HeadlineWriter**  
  - Type: Langchain Agent  
  - Role: Combines prompt and product description to generate headline text  
  - Config: Uses expression to concatenate prompt + product description from form trigger  
  - Inputs: Set_PromptForHeadline (prompt), FormTrigger_CopywritingBrief (product description), LLM_HeadlineWriterModel (model)  
  - Outputs: JSON with generated headline in `output` field  
  - Failure modes: Agent prompt errors, expression evaluation failures, API errors

---

#### 2.3 Evaluation Criteria Creation

**Overview:**  
Automatically defines five scoring criteria with 1-10 scales to assess the headline quality objectively.

**Nodes Involved:**  
- LLM_EvalCriteriaModel  
- Agent_EvalCriteriaBuilder

**Node Details:**

- **LLM_EvalCriteriaModel**  
  - Type: Langchain OpenAI Chat LLM node  
  - Role: Provides GPT-4o-mini model for criteria generation  
  - Config: Same OpenAI API credential, model "gpt-4o-mini"  
  - Inputs: None directly (called by Agent)  
  - Outputs: Model response with suggested criteria  
  - Failure modes: API auth issues, timeouts

- **Agent_EvalCriteriaBuilder**  
  - Type: Langchain Agent  
  - Role: Requests 5 scoring parameters (scale 1-10) to evaluate headline output  
  - Config: Prompt text instructs to create parameters based on headline prompt  
  - Inputs: Prompt from Set_PromptForHeadline, LLM_EvalCriteriaModel  
  - Outputs: Parameters in JSON for use in evaluation  
  - Failure modes: Prompt construction errors, API errors

---

#### 2.4 Headline Scoring and Evaluation

**Overview:**  
Evaluates the generated headline against the criteria, producing detailed scores, an average, and a natural language bottom line.

**Nodes Involved:**  
- LLM_HeadlineEvaluatorModel  
- Agent_HeadlineEvaluator

**Node Details:**

- **LLM_HeadlineEvaluatorModel**  
  - Type: Langchain OpenAI Chat LLM node  
  - Role: GPT-4o-mini model used to score headline  
  - Config: Same OpenAI API credential  
  - Inputs: None directly (used by Agent)  
  - Outputs: Model scoring output to Agent  
  - Failure modes: API errors

- **Agent_HeadlineEvaluator**  
  - Type: Langchain Agent  
  - Role: Assesses headline output using the evaluation criteria; calculates average score and provides bottom line  
  - Config: Complex prompt combining original prompt, headline output, and scoring parameters  
  - Inputs: Headline output (Agent_HeadlineWriter), criteria (Agent_EvalCriteriaBuilder), LLM_HeadlineEvaluatorModel  
  - Outputs: JSON with scores, average, and bottom line text  
  - Failure modes: Expression errors, API errors, malformed output

---

#### 2.5 Iteration Decision and Output Delivery

**Overview:**  
Decides whether the headline needs further improvement or is acceptable. If acceptable, sends the final headline via Gmail.

**Nodes Involved:**  
- LLM_BottomLineModel  
- Agent_IterationDecision  
- If_NeedMoreIterations  
- Send a message

**Node Details:**

- **LLM_BottomLineModel**  
  - Type: Langchain OpenAI Chat LLM node  
  - Role: Reads evaluation output to support iteration decision  
  - Config: GPT-4o-mini model, same credential  
  - Inputs: None directly (used by Agent)  
  - Outputs: Model response fed into decision agent  
  - Failure modes: API timeouts

- **Agent_IterationDecision**  
  - Type: Langchain Agent  
  - Role: Uses bottom line to decide if another iteration is needed; returns "NO" to accept or "YES" with feedback  
  - Config: Prompt uses bottom line from evaluation agent output  
  - Inputs: LLM_BottomLineModel output, Agent_HeadlineEvaluator output  
  - Outputs: String output "NO" or "YES" and feedback  
  - Failure modes: Prompt construction errors, API errors

- **If_NeedMoreIterations**  
  - Type: If node  
  - Role: Checks Agent_IterationDecision output; if "NO" ends workflow by sending email, if "YES" loops back (loop wiring not implemented)  
  - Config: Condition: output equals "NO" (case-sensitive, strict)  
  - Inputs: Agent_IterationDecision  
  - Outputs:  
    - On "NO": proceeds to Send a message node  
    - On "YES": no active loop wiring (currently no action)  
  - Failure modes: Logic errors if output is unexpected

- **Send a message**  
  - Type: Gmail node  
  - Role: Sends the final headline via Gmail using OAuth2 credentials  
  - Config: Message body is the headline output from Agent_HeadlineWriter  
  - Inputs: From If_NeedMoreIterations (on "NO" branch)  
  - Outputs: Email sent confirmation  
  - Failure modes: Gmail OAuth token expiration, SMTP errors

---

### 3. Summary Table

| Node Name                 | Node Type                             | Functional Role                                      | Input Node(s)                   | Output Node(s)                 | Sticky Note                                                                                                                             |
|---------------------------|-------------------------------------|-----------------------------------------------------|--------------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------------------------|
| FormTrigger_CopywritingBrief | Form Trigger                        | Captures product description via web form           | External Webhook               | Set_PromptForHeadline          | 📝 Form Trigger → 🛠️ Set Prompt: captures marketer input                                                                                 |
| Set_PromptForHeadline      | Set                                 | Prepares prompt string for headline generation       | FormTrigger_CopywritingBrief   | Agent_HeadlineWriter           | 🛠️ Prepends system prompt text to input                                                                                                |
| LLM_HeadlineWriterModel    | Langchain OpenAI Chat LLM           | Provides GPT-4o-mini model for headline generation   | Called by Agent_HeadlineWriter | Agent_HeadlineWriter           | 🤖 GPT-4o-mini LLM powers agent                                                                                                       |
| Agent_HeadlineWriter       | Langchain Agent                     | Generates Facebook ad headline                        | Set_PromptForHeadline, LLM_HeadlineWriterModel, FormTrigger_CopywritingBrief | Agent_EvalCriteriaBuilder      | ✍️ Generates headline based on prompt and user input                                                                                   |
| LLM_EvalCriteriaModel      | Langchain OpenAI Chat LLM           | Provides GPT-4o-mini model for criteria generation   | Called by Agent_EvalCriteriaBuilder | Agent_EvalCriteriaBuilder      | 🤖 Same model used to create scoring parameters                                                                                        |
| Agent_EvalCriteriaBuilder  | Langchain Agent                     | Builds 5 scoring criteria (1-10 scale)                | Agent_HeadlineWriter, LLM_EvalCriteriaModel | Agent_HeadlineEvaluator         | 📋 Dynamically creates rubric for headline evaluation                                                                                 |
| LLM_HeadlineEvaluatorModel | Langchain OpenAI Chat LLM           | Provides GPT-4o-mini model for headline evaluation   | Called by Agent_HeadlineEvaluator | Agent_HeadlineEvaluator         | 🤖 Evaluates headline using criteria                                                                                                 |
| Agent_HeadlineEvaluator    | Langchain Agent                     | Scores headline and returns numeric + bottom line    | Agent_EvalCriteriaBuilder, LLM_HeadlineEvaluatorModel | Agent_IterationDecision        | 🔍 Produces scores + average + plain language bottom line                                                                             |
| LLM_BottomLineModel        | Langchain OpenAI Chat LLM           | Supports iteration decision based on evaluation output | Called by Agent_IterationDecision | Agent_IterationDecision        | 🤖 Reads evaluator output for decision making                                                                                        |
| Agent_IterationDecision    | Langchain Agent                     | Decides if headline needs iteration ("YES"/"NO")     | Agent_HeadlineEvaluator, LLM_BottomLineModel | If_NeedMoreIterations          | 🟢/🔴 Returns NO to accept or YES + feedback for iteration                                                                             |
| If_NeedMoreIterations      | If node                            | Routes flow based on iteration decision               | Agent_IterationDecision        | Send a message (NO branch)     | 🔀 Routes NO → email send; YES → loop back (loop not wired)                                                                            |
| Send a message             | Gmail node                        | Sends final approved headline by email                | If_NeedMoreIterations          | None                          | Sends headline via Gmail using OAuth2                                                                                                 |
| Sticky Note9               | Sticky Note                       | Workflow assistance contact and links                 | None                         | None                          | Contact: Yaron@nofluff.online; YouTube & LinkedIn links                                                                               |
| Sticky Note4               | Sticky Note                       | Full workflow explanation and section details         | None                         | None                          | Detailed explanation of each section, node purposes, and benefits                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node**  
   - Type: Form Trigger  
   - Configure webhook with any unique ID  
   - Add one required form field: "What is your product about?" (text)  

2. **Create Set Node for Prompt Preparation**  
   - Type: Set  
   - Add a field named "Prompt For Agent" (string) with value:  
     `Write a facebook ad headline for this product:`  
   - Connect Form Trigger output → Set node input  

3. **Create Langchain OpenAI Chat LLM Node for Headline Generation**  
   - Type: Langchain LLM Chat (OpenAI)  
   - Set model to "gpt-4o-mini"  
   - Assign OpenAI API credential with valid key  
   - No direct input connections  

4. **Create Langchain Agent Node for Headline Writer**  
   - Type: Langchain Agent  
   - Set prompt expression:  
     `={{ $json['Prompt For Agent'] }}:{{ $('FormTrigger_CopywritingBrief').item.json['What is your product about?'] }}`  
   - Assign the LLM Headline Writer Model node as its language model  
   - Connect Set_PromptForHeadline → Agent_HeadlineWriter  
   - Connect LLM_HeadlineWriterModel → Agent_HeadlineWriter (AI language model input)  

5. **Create Langchain OpenAI Chat LLM Node for Eval Criteria**  
   - Same setup as HeadlineWriterModel but separate node  
   - Model: gpt-4o-mini  
   - Same OpenAI credential  

6. **Create Langchain Agent Node for Eval Criteria Builder**  
   - Type: Langchain Agent  
   - Prompt text:  
     ```
     Your goal is to define parameters for assessing the output of this prompt:
     {{ $('Set_PromptForHeadline').item.json['Prompt For Agent'] }}

     Which parameters would you suggest? Give me 5 parameters including the scale. All should be in scale of 1-10 (10 will be the most positive)
     ```  
   - Connect Agent_HeadlineWriter output → Agent_EvalCriteriaBuilder input  
   - Connect LLM_EvalCriteriaModel → Agent_EvalCriteriaBuilder (AI language model input)  

7. **Create Langchain OpenAI Chat LLM Node for Headline Evaluation**  
   - Model: gpt-4o-mini  
   - Same credentials  

8. **Create Langchain Agent Node for Headline Evaluator**  
   - Prompt text:  
     ```
     Your goal is to assess the output of an AI agent.
     It was supposed to:
     {{ $('Set_PromptForHeadline').item.json['Prompt For Agent'] }}

     The answer it gave is:{{ $('Agent_HeadlineWriter').item.json.output }}

     Provide your response using these parameters:{{ $json.output }}. Make sure you calculate your average and give a bottom line.
     ```  
   - Connect Agent_EvalCriteriaBuilder → Agent_HeadlineEvaluator  
   - Connect LLM_HeadlineEvaluatorModel → Agent_HeadlineEvaluator  

9. **Create Langchain OpenAI Chat LLM Node for Bottom Line Model**  
   - Model: gpt-4o-mini  
   - Same credentials  

10. **Create Langchain Agent Node for Iteration Decision**  
    - Prompt text:  
      ```
      Based on the bottom line:{{ $json.output }}

      Decide if we need another iteration.
      Return NO if we don't,
      otherwise return YES and add your feedback.
      ```  
    - Connect Agent_HeadlineEvaluator → Agent_IterationDecision  
    - Connect LLM_BottomLineModel → Agent_IterationDecision  

11. **Create If Node for Iteration Check**  
    - Condition: output of Agent_IterationDecision equals "NO" (case sensitive, strict)  
    - Connect Agent_IterationDecision → If_NeedMoreIterations  

12. **Create Gmail Node to Send Final Headline**  
    - Configure with Gmail OAuth2 credentials  
    - Message body: `={{ $('Agent_HeadlineWriter').item.json.output }}`  
    - Connect If_NeedMoreIterations (NO branch) → Send a message node  

13. **(Optional) Looping Setup**  
    - For YES output of If_NeedMoreIterations, connect back to Agent_HeadlineWriter to allow iteration (not currently wired)  

14. **Add Sticky Notes for Documentation**  
    - Add workflow assistance and detailed explanations as per original notes  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                   | Context or Link                                                                                       |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------|
| Workflow assistance contact: Yaron@nofluff.online. For tips and tutorials, visit YouTube channel: https://www.youtube.com/@YaronBeen/videos and LinkedIn: https://www.linkedin.com/in/yaronbeen/                                                                | Contact and educational resources for workflow support                                              |
| This workflow automates Facebook ad headline generation and quality scoring using GPT-4o-mini and n8n agents. It features modular design and objective criteria generation for repeatable quality checks. Iteration logic ensures only high-quality headlines proceed. | Branding and functional summary                                                                      |
| To extend the workflow, add storage (e.g., Google Sheets) or additional notification nodes after the iteration decision node. Loop limits should be implemented to avoid infinite loops in real use.                                                           | Implementation advice                                                                                 |

---

**Disclaimer:** The text provided originates exclusively from an automated workflow created with n8n, a tool for integration and automation. This process strictly complies with current content policies and contains no illegal, offensive, or protected material. All data processed is legal and public.