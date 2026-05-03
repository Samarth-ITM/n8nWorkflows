Iterative Content Refinement with GPT-4 Multi-Agent Feedback System

https://n8nworkflows.xyz/workflows/iterative-content-refinement-with-gpt-4-multi-agent-feedback-system-5597


# Iterative Content Refinement with GPT-4 Multi-Agent Feedback System

# 1. Workflow Overview

This n8n workflow, titled **"Iterative Content Refinement with GPT-4 Multi-Agent Feedback System"**, is designed to generate, critique, and iteratively improve branding ideas (product names and taglines) using a multi-agent loop powered by GPT-4. It automates a creative feedback loop where initial ideas are generated, critically analyzed, refined, and evaluated repeatedly until satisfactory output or a maximum iteration count is reached.

The workflow is well-suited for use cases such as branding agencies, marketing teams, product managers, or creative professionals who want systematic, AI-driven iterative enhancement of creative content.

The workflow's logic is organized into the following key blocks:

- **1.1 Input Initialization and Trigger**  
  Starts the workflow and initializes iteration variables.

- **1.2 Initial Idea Generation**  
  Generates initial branding ideas using the AI Agent.

- **1.3 Iterative Feedback Loop**  
  Runs multi-turn loop consisting of:  
  - Critic Agent analyzing ideas,  
  - Refiner Agent improving ideas based on critique,  
  - Evaluation Agent ranking and deciding completion.

- **1.4 Loop Control and Data Management**  
  Controls iteration count and exit conditions, preparing data for next loop or final output.

- **1.5 Output Parsing and Finalization**  
  Parses AI outputs structurally and formats final data.

- **1.6 Setup and Metadata**  
  Includes workflow trigger and extensive setup instructions via sticky note.

# 2. Block-by-Block Analysis

---

## 2.1 Input Initialization and Trigger

**Overview:**  
This block triggers the workflow manually and sets initial data fields to track ideas, iteration turns, and completion status.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Edit Fields

### Node Details

**When clicking ‘Execute workflow’**  
- Type: Manual Trigger  
- Role: Initiates workflow execution on user demand.  
- Configuration: Default, no parameters.  
- Inputs: None  
- Outputs: Starts flow to AI Agent node  
- Potential Failures: None typical; ensure manual activation.

**Edit Fields**  
- Type: Set node  
- Role: Initializes key workflow variables:  
  - `ideas` (array) to hold generated ideas  
  - `turn` (number) to track iteration count (default 1)  
  - `done` (string) to track completion status ("No" default)  
- Configuration: Sets fields based on input or defaults when undefined.  
- Input: From Loop Over Items (in later iterations), or initial trigger  
- Output: To If node for loop control  
- Edge Cases: Misconfiguration of default values could stall loop.

---

## 2.2 Initial Idea Generation

**Overview:**  
Generates 3 initial branding name + tagline options based on a provided product description.

**Nodes Involved:**  
- AI Agent  
- Loop Over Items

### Node Details

**AI Agent**  
- Type: LangChain Agent (GPT-4 based)  
- Role: Generates 3 distinct name + tagline combos from product description.  
- Configuration:  
  - Prompt includes product description from input JSON field `productDescription`.  
  - System message guides agent to return output as JSON array with `name` and `tagline` fields per item.  
- Input: From manual trigger (with productDescription set externally)  
- Output: To Loop Over Items node  
- Edge Cases: If `productDescription` is empty or malformed, output may be invalid.

**Loop Over Items**  
- Type: SplitInBatches  
- Role: Controls batch processing for items; here, used as part of iterative loop mechanism.  
- Configuration: Reset enabled to loop over the data repeatedly.  
- Input: From AI Agent or Edit Fields1 (loop continuation)  
- Output: To Edit Fields (loop control)  
- Potential Failures: Misconfiguration could cause infinite loops or batch stalls.

---

## 2.3 Iterative Feedback Loop

**Overview:**  
This is the core loop performing critique, refinement, and evaluation of the branding ideas. The loop repeats until max turns or satisfactory completion.

**Nodes Involved:**  
- If  
- Critic Agent  
- Refiner Agent  
- Evaluation agent  
- Edit Fields (within loop)  
- Code (manages iteration increment)  
- Edit Fields1 (prepares data for next loop)

### Node Details

**If**  
- Type: Conditional  
- Role: Checks if loop should continue or exit. Conditions:  
  - `turn` equals 5 (max iteration), OR  
  - `done` equals "Yes" (completion)  
- Input: From Edit Fields  
- Output:  
  - TRUE branch: ends loop (no next node)  
  - FALSE branch: proceeds to Critic Agent  
- Edge Cases: Incorrect conditions could cause premature exit or infinite loops.

**Critic Agent**  
- Type: LangChain Agent  
- Role: Acts as a critical branding analyst, reviews current ideas and provides flaw summaries and improvement suggestions.  
- Configuration:  
  - Receives ideas from Edit Fields node.  
  - System prompt instructs to provide concise flaw summary and suggestions.  
- Input: From If node (FALSE branch)  
- Output: To Refiner Agent  
- Edge Cases: May produce unclear critiques if input ideas are malformed.

**Refiner Agent**  
- Type: LangChain Agent  
- Role: Uses original ideas and critic feedback to produce 3 improved name + tagline pairs.  
- Configuration:  
  - Inputs include original ideas and critic feedback.  
  - System prompt instructs concise, clever, impactful refinements.  
- Input: From Critic Agent  
- Output: To Evaluation agent  
- Edge Cases: Risk of repetitive ideas if feedback is suboptimal.

**Evaluation agent**  
- Type: LangChain Agent with Output Parser  
- Role: Ranks the refined ideas and decides if the output is satisfactory (`done` field).  
- Configuration:  
  - Receives refined ideas and current turn count.  
  - System prompt instructs to output JSON with ranked ideas, `done` status ("Yes" or "No"), and `turn`.  
  - Uses Structured Output Parser node for JSON validation.  
- Input: From Refiner Agent and Structured Output Parser  
- Output: To Code node  
- Edge Cases: Misparsed JSON could break iteration data flow.

**Code**  
- Type: JavaScript Code  
- Role: Increments the `turn` count, preserves ideas and `done` status for next loop iteration.  
- Configuration: Reads from Evaluation agent output and prepares JSON for next iteration.  
- Input: From Evaluation agent  
- Output: To Edit Fields1  
- Edge Cases: Coding errors could reset or corrupt iteration data.

**Edit Fields1**  
- Type: Set node  
- Role: Sets `output` (ideas), `turn`, and `done` fields from Code node for Loop Over Items node to process next iteration.  
- Configuration: Straight mapping of iteration state fields.  
- Input: From Code node  
- Output: To Loop Over Items  
- Edge Cases: Misalignment of fields would break loop continuation.

---

## 2.4 Loop Control and Data Management

**Overview:**  
Controls the iteration loop and data flow between iterations to ensure consistent state updates and loop exit conditions.

**Nodes Involved:**  
- Loop Over Items  
- Edit Fields  
- If  
- Code  
- Edit Fields1

### Node Details

**Loop Over Items**  
- Role: Controls batch iteration, resets when needed to loop until conditions met.  
- Input/Output: Connects initial AI Agent output and subsequent iteration states.

**Edit Fields**  
- Role: Prepares and tracks iteration variables for the If node to evaluate loop continuation.

**If**  
- Role: Decides whether to continue the loop based on iteration count and done status.

**Code**  
- Role: Manages iteration increment and data packaging for next cycle.

**Edit Fields1**  
- Role: Prepares iteration data for next loop cycle.

---

## 2.5 Output Parsing and Finalization

**Overview:**  
Ensures AI-generated outputs are correctly parsed and structured for the evaluation agent to process and for subsequent nodes to consume.

**Nodes Involved:**  
- Structured Output Parser  
- Evaluation agent (has output parser attached)

### Node Details

**Structured Output Parser**  
- Type: LangChain Structured Output Parser  
- Role: Validates that AI outputs comply with expected JSON schema for ideas, done status, and turn count.  
- Configuration: Uses example JSON schema with fields for `ideas` array, `done` string, and `turn` number.  
- Input: From Evaluation agent raw output  
- Output: Parsed JSON to Evaluation agent for final processing  
- Edge Cases: Parsing errors if AI outputs deviate from schema.

---

## 2.6 Setup and Metadata

**Overview:**  
Provides manual trigger and extensive setup instructions for users to customize and maintain the workflow, including credential requirements.

**Nodes Involved:**  
- When clicking ‘Execute workflow’ (Manual Trigger)  
- Sticky Note

### Node Details

**Sticky Note**  
- Contains detailed setup guide, prerequisites, configuration steps for each AI agent and node, and notes on loop control.  
- Contains author credit: Sebastian/OptiLever [LinkedIn](www.linkedin.com/in/sebastian-9ab9b9242)  
- Emphasizes connecting OpenAI API credentials and provides stepwise customization instructions.

---

# 3. Summary Table

| Node Name               | Node Type                      | Functional Role                      | Input Node(s)                 | Output Node(s)                | Sticky Note                                                                                                     |
|-------------------------|--------------------------------|------------------------------------|------------------------------|------------------------------|----------------------------------------------------------------------------------------------------------------|
| When clicking ‘Execute workflow’ | Manual Trigger                 | Starts workflow                    | None                         | AI Agent                     | See setup guide detailing trigger and initial configuration                                                    |
| AI Agent                | LangChain Agent                | Generates initial ideas            | When clicking ‘Execute workflow’ | Loop Over Items              |                                                                                                                |
| Loop Over Items         | SplitInBatches                 | Controls iterative batch processing | AI Agent, Edit Fields1        | Edit Fields                  |                                                                                                                |
| Edit Fields             | Set                           | Initializes/tracks iteration data  | Loop Over Items               | If                           |                                                                                                                |
| If                      | Conditional                   | Loops control: max turns or done   | Edit Fields                  | Critic Agent (FALSE branch)  |                                                                                                                |
| Critic Agent            | LangChain Agent                | Critiques current ideas            | If (FALSE branch)             | Refiner Agent                |                                                                                                                |
| Refiner Agent           | LangChain Agent                | Refines ideas based on critique    | Critic Agent                 | Evaluation agent             |                                                                                                                |
| Evaluation agent        | LangChain Agent + Output Parser | Ranks refined ideas, decides completion | Refiner Agent, Structured Output Parser | Code                      |                                                                                                                |
| Structured Output Parser | LangChain Output Parser       | Parses AI JSON outputs             | Evaluation agent (raw output) | Evaluation agent             |                                                                                                                |
| Code                    | Code                          | Increments turn, packages data     | Evaluation agent             | Edit Fields1                 |                                                                                                                |
| Edit Fields1            | Set                           | Prepares data for next loop cycle  | Code                         | Loop Over Items              |                                                                                                                |
| Sticky Note             | Sticky Note                   | Setup instructions and metadata    | None                         | None                        | See detailed workflow setup instructions, author credit, and OpenAI credential reminder                       |

# 4. Reproducing the Workflow from Scratch

1. **Create Manual Trigger Node**  
   - Type: `Manual Trigger`  
   - No parameters necessary.  
   - This node starts the workflow manually.

2. **Create AI Agent Node (Initial Idea Generator)**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Prompt: `Product: {{$json["productDescription"]}}`  
   - System Message: "You are a creative branding agent. Based on this product description, generate 3 distinct name + tagline options. Return as JSON array with objects containing `name` and `tagline`."  
   - Connect Manual Trigger output to this node’s input.

3. **Create SplitInBatches Node (Loop Over Items)**  
   - Type: `SplitInBatches`  
   - Enable `reset` option to allow looping.  
   - Connect AI Agent output to this node.

4. **Create Set Node (Edit Fields)**  
   - Type: `Set`  
   - Fields to set:  
     - `ideas` (type: Array) → from `$json.output` or empty array initially  
     - `turn` (type: Number) → default to 1 if undefined  
     - `done` (type: String) → default to "No" if undefined  
   - Connect Loop Over Items output to this node.

5. **Create If Node**  
   - Type: `If`  
   - Conditions:  
     - Numeric Equals: `$json.turn == 5` (max iterations reached)  
     - String Equals: `$json.done == "Yes"` (completion)  
   - Connect Edit Fields output to If node.

6. **Create Critic Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Input text expression: `Ideas:\n{{JSON.stringify($('Edit Fields').item.json.ideas)}}`  
   - System Message: "You are a critical branding analyst. For each name + tagline below, provide a short flaw summary and suggestion for improvement."  
   - Connect If node’s FALSE output (loop continues) to this node.

7. **Create Refiner Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Input text expression:  
     ```
     **Original Ideas:** {{ JSON.stringify($('Edit Fields').item.json.ideas) }}

     **Critics Feedback:** {{ $json.output }}
     ```  
   - System Message: "You are a refiner. Use the original ideas and critic’s feedback to create 3 improved name + tagline combos. Be concise, clever, impactful. Return JSON array format."  
   - Connect Critic Agent output to Refiner Agent.

8. **Create Structured Output Parser Node**  
   - Type: `@n8n/n8n-nodes-langchain.outputParserStructured`  
   - Provide example JSON schema for validation:  
     ```
     {
       "ideas": [
         { "name": "Name1", "tagline": "Tagline1" },
         { "name": "Name2", "tagline": "Tagline2" },
         { "name": "Name3", "tagline": "Tagline3" }
       ],
       "done": "Yes",
       "turn": 2
     }
     ```  
   - Connect Refiner Agent output to Evaluation agent through this node.

9. **Create Evaluation Agent Node**  
   - Type: `@n8n/n8n-nodes-langchain.agent` with output parser enabled  
   - Input text expression:  
     ```
     **Results To Examine:** {{ $json.output }}

     **Turns Through Loop:** {{ $('Edit Fields').item.json.turn }}
     ```  
   - System Message: "You are an evaluator. Rank the 3 improved name ideas and taglines. Recommend if names/taglines are good enough to make anyone want to purchase (Yes/No). Output JSON with `ideas`, `done`, and `turn`."  
   - Connect Structured Output Parser output to this node.

10. **Create Code Node**  
    - Type: `Code` (JavaScript)  
    - JS code to increment turn and preserve data:  
      ```js
      const ideas = $json.output.ideas || [];
      const currentTurn = $json.output.turn ?? 0;
      const currentDone = $json.output.done || "No";

      const nextTurn = currentTurn + 1;

      return [
        {
          json: {
            ideas,
            turn: nextTurn,
            done: currentDone
          }
        }
      ];
      ```  
    - Connect Evaluation Agent output to this node.

11. **Create Set Node (Edit Fields1)**  
    - Type: `Set`  
    - Fields:  
      - `output` (array) ← `{{$json.ideas}}`  
      - `turn` (number) ← `{{$json.turn}}`  
      - `done` (string) ← `{{$json.done}}`  
    - Connect Code node output to this node.

12. **Loop Back to SplitInBatches**  
    - Connect Edit Fields1 output back to Loop Over Items node to continue or end the loop.

13. **Connect If Node TRUE branch**  
    - When loop exit condition met, no further nodes connected, or connect to downstream processing as desired.

14. **Credential Setup**  
    - Configure OpenAI API credentials in all LangChain agent nodes and OpenAI Chat Model if used.  
    - Use OAuth2 or API key as per OpenAI account.

15. **Optional: Add Sticky Note**  
    - Add a Sticky Note node with setup instructions, author credit, and links as metadata for users.

# 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                    | Context or Link                               |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------|
| Workflow author: Sebastian/OptiLever. LinkedIn profile: www.linkedin.com/in/sebastian-9ab9b9242                                                                                                                                                                                  | Author credit                                |
| Prerequisite: Connect OpenAI API account with appropriate credentials for GPT-4 access.                                                                                                                                                                                        | Credential setup                             |
| Setup guidance includes initializing `ideas` as array, `turn` as iteration count, and `done` flag for loop control.                                                                                                                                                            | Workflow customization                       |
| The iterative loop is capped at 5 turns to avoid infinite cycles but can exit early if results are satisfactory.                                                                                                                                                                 | Loop control                                |
| The multi-agent approach includes specialized agents: Creative generator, Critic, Refiner, and Evaluator for comprehensive content refinement.                                                                                                                                 | Workflow design rationale                    |
| Structured Output Parser validates AI JSON output to mitigate parsing errors and improve workflow robustness.                                                                                                                                                                    | Output validation                           |
| For more info on LangChain nodes and prompts, see n8n documentation: https://docs.n8n.io/integrations/built-in-nodes/n8n-nodes-langchain/                                                                                                                                        | Official n8n LangChain integration docs     |
| The workflow is not active by default (`active: false`), so users must enable it for execution.                                                                                                                                                                                 | Execution note                              |

---

*Disclaimer:* The text provided is exclusively from an automated workflow created with n8n, an integration and automation tool. The processing strictly follows current content policies and contains no illegal, offensive, or protected elements. All manipulated data is legal and public.