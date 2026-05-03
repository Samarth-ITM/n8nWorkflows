Simulate Debates Between AI Agents Using Mistral to Optimize Answers

https://n8nworkflows.xyz/workflows/simulate-debates-between-ai-agents-using-mistral-to-optimize-answers-5682


# Simulate Debates Between AI Agents Using Mistral to Optimize Answers

### 1. Workflow Overview

This workflow, titled **"Simulate Debates Between AI Agents Using Mistral to Optimize Answers"**, orchestrates a controlled debate among multiple AI agents with distinct personalities and viewpoints. Its main purpose is to simulate a discussion environment where AI agents collaboratively refine and optimize an input text through multiple debate rounds, ultimately generating a more nuanced and improved final output. This approach is valuable when a single AI response might be insufficiently nuanced or when a scenario requires simulating interactions such as meetings, interviews, storytelling, or conferences.

**Target Use Cases:**
- Meeting Simulation
- Interview Simulation
- Storywriter Test Environment
- Forum, Conference, or Symposium Simulation

**Logical Blocks:**

- **1.1 Initialization & Input Configuration**: Setting up triggers, workflow arguments, and initial input state.
- **1.2 Round Execution (AI Agent Debate)**: Running each AI agent's turn individually and gathering their structured responses.
- **1.3 Round Results Aggregation & Environment Processing**: Collecting all AI responses in a round and synthesizing a collective result and summary.
- **1.4 Loop & Round Management**: Controlling the iteration over rounds and updating inputs for the next round.
- **1.5 Termination & Output Delivery**: Ending the debate once all rounds are completed and delivering the final optimized input.

---

### 2. Block-by-Block Analysis

#### 2.1 Initialization & Input Configuration

- **Overview:**  
  This block sets up the workflow trigger, loads predefined workflow arguments (including AI agent profiles and environment settings), and prepares the initial input and debate state.

- **Nodes Involved:**  
  - Schedule  
  - When clicking ‘Execute workflow’ (Manual Trigger)  
  - Configure Workflow Args (Global Constants Node)  
  - Prepare Input (Set)  
  - Guarantee Input (Set)  
  - Email Trigger (IMAP) [Disabled]  
  - Sticky Notes (various for documentation)

- **Node Details:**

  - **Schedule**  
    - Type: Schedule Trigger  
    - Configured to trigger weekly on Monday, Tuesday, Wednesday, Thursday, and Friday at 12:00 PM.  
    - Potential failure: Timezone mismatches, missed triggers.

  - **When clicking ‘Execute workflow’**  
    - Type: Manual Trigger  
    - Allows manual start of the workflow for testing or on-demand runs.

  - **Configure Workflow Args**  
    - Type: Global Constants Node (custom "n8n-nodes-globals")  
    - Loads all key constants and variables such as input text, scenario description, rounds, AI agents' configurations, and AI environment settings.  
    - Uses credentials named "Debate Arena V1".  
    - Edge cases: Missing or invalid global constants, credential failures.

  - **Prepare Input**  
    - Type: Set  
    - Initializes variables:  
      - `input` from global constants  
      - `current_round` = 1  
      - `round_summary` = "The debate begins."  
      - `round_result` = "This is the first round. No last round result yet."  
    - Passes all other incoming fields unchanged.  
    - Inputs from Configure Workflow Args; outputs to Guarantee Input.

  - **Guarantee Input**  
    - Type: Set  
    - Ensures consistent input state for each round by consolidating:  
      - `input` (current input text)  
      - `current_round` (the debate round number)  
      - `constants` (all workflow args from Configure Workflow Args)  
      - `round_summary` and `round_result` from previous step or iteration.  
    - Inputs from Prepare Input or Update Input.

  - **Email Trigger (IMAP)** [Disabled]  
    - Type: Email Read IMAP  
    - Intended for triggering workflow by email with specific subject filters. Disabled in this instance.

  - **Sticky Notes**  
    - Provide detailed documentation and instructions inside the workflow for user guidance.  
    - Significantly help understand arguments structure, example inputs, and workflow purpose.

- **Input/Output Connections:**  
  - Schedule and Manual Trigger feed into Configure Workflow Args.  
  - Configure Workflow Args outputs to Prepare Input -> Guarantee Input.  
  - Guarantee Input outputs to Round Loop (start of next block).

- **Version-Specific Requirements:**  
  - Uses custom node "n8n-nodes-globals" v1.1.0 for global constants.  
  - Tested on n8n v1.100.1.

- **Potential Failure Types:**  
  - Credential failures for global constants.  
  - Missing or malformed workflow arguments.  
  - Schedule misconfiguration or timezone issues.

---

#### 2.2 Round Execution (AI Agent Debate)

- **Overview:**  
  Runs each AI agent’s turn individually per round. Splits the AI agents list, feeds each agent their input and environment context, and retrieves their structured JSON reply.

- **Nodes Involved:**  
  - Round Loop (Split in Batches, batch size 1)  
  - Split Out AI Agents (Split Out node)  
  - Wait Before Sending Agents (Wait node)  
  - Debate Actor Abstraction (LangChain Agent node)  
  - Mistral Cloud Chat Model (AI Language Model)  
  - JSON Output Parser (Structured Output Parser)  
  - Wait 1 and Wait 2 (Wait nodes)  

- **Node Details:**

  - **Round Loop**  
    - Type: Split In Batches  
    - Batch size 1 to process one AI agent at a time per round.  
    - Resets after each batch.  
    - Inputs from Guarantee Input.  
    - Outputs to Split Out AI Agents.

  - **Split Out AI Agents**  
    - Type: Split Out  
    - Splits the JSON array of AI agents from workflow constants into individual items.  
    - Output goes into Wait Before Sending Agents.

  - **Wait Before Sending Agents**  
    - Type: Wait  
    - No specified delay; used as a control point before sending requests to AI agents.  
    - Outputs to Debate Actor Abstraction.

  - **Debate Actor Abstraction**  
    - Type: LangChain Agent node  
    - Uses the Mistral Cloud Chat Model to simulate the AI agent debating.  
    - Inputs: AI agent profile, current input, round summary, last round result.  
    - Prompt system message uses agent properties like name, role, will, reason, likes, dislikes, and writing style.  
    - Expects JSON-formatted output with keys: ai_name, ai_status, ai_targets, reply, cause_for_reply.  
    - Retry policy: 2 tries, 3-second wait between attempts.  
    - Outputs parsed JSON to JSON Output Parser.  
    - Potential failures: API errors, JSON parsing errors, rate limits.

  - **Mistral Cloud Chat Model**  
    - Type: LangChain Language Model node  
    - Uses "mistral-small-latest" model with safe mode enabled and max 2 retries.  
    - Credential: Mistral Cloud API (blank account placeholder).  
    - Provides LLM completions to Debate Actor Abstraction.

  - **JSON Output Parser**  
    - Type: LangChain Structured Output Parser  
    - Validates and parses the JSON output from Debate Actor Abstraction according to strict schema.  
    - Ensures AI agent replies are structured and usable downstream.

  - **Wait 1 and Wait 2**  
    - Type: Wait nodes  
    - Introduce 1 second delays to manage pacing and API rate limits.

- **Input/Output Connections:**  
  - Guarantee Input -> Round Loop -> Split Out AI Agents -> Wait Before Sending Agents -> Debate Actor Abstraction -> JSON Output Parser -> Wait 1 -> Wait 2 -> Debate Loop.

- **Version-Specific Requirements:**  
  - LangChain nodes require n8n v1.100.1+ for compatibility.  
  - Mistral Cloud API credentials configured.

- **Potential Failure Types:**  
  - AI model API failures or timeouts.  
  - Output parsing failures if AI response is malformed.  
  - Exceeding rate limits requiring waits or backoff.

---

#### 2.3 Round Results Aggregation & Environment Processing

- **Overview:**  
  Aggregates all AI agents’ replies of the current round, and runs a specialized AI environment agent to produce a round summary, round result, and rewritten input based on combined AI agent inputs.

- **Nodes Involved:**  
  - Debate Loop (Split In Batches)  
  - Aggregate (Aggregate node)  
  - Debate Environment (LangChain Agent node)  
  - Mistral Cloud Chat Model 2 (AI Language Model)  
  - JSON Output Parser 2 (Structured Output Parser)  
  - Simple Memory 2 (Memory Buffer Window) [Disabled]  
  - If No More Rounds (If node)  
  - End of Debate (No Op node)  

- **Node Details:**

  - **Debate Loop**  
    - Type: Split In Batches  
    - Batch size equals number of AI agents (e.g., 3).  
    - Controls the processing of entire rounds after all agents have replied.  
    - Outputs to Aggregate.

  - **Aggregate**  
    - Type: Aggregate All Item Data  
    - Collects all AI agent replies into a single data property array for environment processing.  
    - Outputs to Debate Environment.

  - **Debate Environment**  
    - Type: LangChain Agent node  
    - Uses Mistral Cloud Chat Model 2 to process the entire round’s AI agent replies.  
    - System message and rewrite goal are taken from workflow args' AI environment context.  
    - Produces structured JSON output with: round_summary, round_result, rewritten_input.  
    - Retry policy: 2 tries, 3-second wait between attempts.  
    - Outputs parsed JSON to JSON Output Parser 2.

  - **Mistral Cloud Chat Model 2**  
    - Same configuration as the first Mistral node, but dedicated for environment synthesis.

  - **JSON Output Parser 2**  
    - Validates output from Debate Environment node, ensuring data integrity for round summary, results, and rewritten input.

  - **Simple Memory 2** [Disabled]  
    - Intended to maintain conversational memory for the environment; disabled here.

  - **If No More Rounds**  
    - Conditional node checks if `current_round` equals configured number of rounds.  
    - If true, proceeds to End of Debate; else, continues updating input for next round.

  - **End of Debate**  
    - No operation node, marks end of workflow execution upon completion of all rounds.

- **Input/Output Connections:**  
  - Debate Actor Abstraction -> Wait 2 -> Debate Loop -> Aggregate -> Debate Environment -> JSON Output Parser 2 -> If No More Rounds -> End of Debate / Update Input.

- **Version-Specific Requirements:**  
  - LangChain nodes and Mistral API credentials as above.

- **Potential Failure Types:**  
  - Model API failures when aggregating inputs.  
  - JSON parsing or schema validation errors.  
  - Logic errors in round count comparisons.

---

#### 2.4 Loop & Round Management

- **Overview:**  
  Controls the iterative process of the debate rounds, updating the input text and round counters after each round, and feeding the updated input back into the debate execution cycle.

- **Nodes Involved:**  
  - Update Input (Set)  
  - Guarantee Input (Set)  
  - Round Loop (Split In Batches)  
  - Debate Loop (Split In Batches)  
  - If No More Rounds (If)  

- **Node Details:**

  - **Update Input**  
    - Type: Set  
    - Updates debate variables after each round:  
      - `input` is set to the newly rewritten input from Debate Environment output.  
      - `current_round` incremented by 1.  
      - Updates `round_summary` and `round_result` properties.  
    - Outputs to Guarantee Input node to reset state for next round.

  - **Guarantee Input**  
    - As described in block 2.1, reused here to maintain state consistency throughout rounds.

  - **Round Loop & Debate Loop**  
    - Manage per-agent and per-round batching and execution as described above.

  - **If No More Rounds**  
    - Determines if debate should continue or end.

- **Input/Output Connections:**  
  - If No More Rounds (false) outputs to Update Input -> Guarantee Input -> Round Loop.

- **Version-Specific Requirements:**  
  - None additional; uses built-in Split In Batches and Set nodes.

- **Potential Failure Types:**  
  - Off-by-one errors in round counting.  
  - Data corruption in input update.

---

#### 2.5 Termination & Output Delivery

- **Overview:**  
  Finalizes the workflow execution and provides the optimized input after all debate rounds have completed.

- **Nodes Involved:**  
  - End of Debate (No Op)  
  - Output is the last updated input containing:  
    - `round_result`  
    - `round_summary`  
    - `rewritten_input`

- **Node Details:**

  - **End of Debate**  
    - Type: No Operation  
    - Acts as a logical endpoint to stop execution cleanly.

  - **Final Output**  
    - The final output JSON contains the optimized input text and summaries from the debate rounds.

- **Input/Output Connections:**  
  - If No More Rounds (true) outputs here.

- **Potential Failure Types:**  
  - None significant; end of process.

---

### 3. Summary Table

| Node Name                | Node Type                                   | Functional Role                              | Input Node(s)              | Output Node(s)                 | Sticky Note                                                                                                        |
|--------------------------|---------------------------------------------|----------------------------------------------|----------------------------|-------------------------------|--------------------------------------------------------------------------------------------------------------------|
| Schedule                 | Schedule Trigger                            | Start workflow on schedule                    |                            | Configure Workflow Args        | Setup and Start: Schedule triggers workflow at configured days and time.                                           |
| When clicking ‘Execute workflow’ | Manual Trigger                         | Manual start of workflow                       |                            | Configure Workflow Args        | Setup and Start: Manual trigger for on-demand execution.                                                            |
| Configure Workflow Args  | Global Constants (n8n-nodes-globals)       | Load workflow constants & AI configuration    | Schedule, Manual Trigger    | Prepare Input                 | Useful for quickly setting variables for new workflow variants. See notes on Args configuration.                    |
| Prepare Input            | Set                                         | Initialize debate variables for round 1       | Configure Workflow Args     | Guarantee Input               | Initializes input text, round counters, and summaries.                                                             |
| Guarantee Input          | Set                                         | Ensure consistent state per round              | Prepare Input, Update Input | Round Loop                   | Maintains input, round info, and constants for each loop iteration.                                                |
| Email Trigger (IMAP)     | Email Read IMAP (disabled)                   | Optional trigger via email                      |                            | Configure Workflow Args        | Disabled. Intended for email activation with subject filters.                                                      |
| Round Loop               | Split In Batches                             | Process one AI agent at a time per round       | Guarantee Input            | Split Out AI Agents           | Controls per-agent execution within a round.                                                                        |
| Split Out AI Agents      | Split Out                                    | Split AI agents list into individual items    | Round Loop                 | Wait Before Sending Agents    | Splits AI agents for individual processing.                                                                         |
| Wait Before Sending Agents | Wait                                        | Control pacing before agent execution          | Split Out AI Agents        | Debate Actor Abstraction      | Controls timing before sending agent inputs.                                                                        |
| Debate Actor Abstraction | LangChain Agent                              | Execute AI agent turn with specific profile    | Wait Before Sending Agents | JSON Output Parser            | Runs AI agents via Mistral model, expects JSON output.                                                              |
| Mistral Cloud Chat Model | LangChain Language Model                     | Provide LLM completions for AI agents          | Debate Actor Abstraction   | Debate Actor Abstraction      | Mistral model "mistral-small-latest" with safe mode enabled.                                                        |
| JSON Output Parser       | LangChain Structured Output Parser           | Validate and parse AI agent JSON output        | Debate Actor Abstraction   | Wait 1                       | Ensures structured AI agent responses.                                                                              |
| Wait 1                   | Wait                                         | Delay control                                  | JSON Output Parser         | Wait 2                       | Adds 1 second delay.                                                                                                |
| Wait 2                   | Wait                                         | Delay control                                  | Wait 1                     | Debate Loop                  | Adds 1 second delay.                                                                                                |
| Debate Loop              | Split In Batches                             | Aggregate all AI agent turns per round         | Wait 2                     | Aggregate                    | Controls end-of-round processing after all agents have replied.                                                    |
| Aggregate                | Aggregate All Items Data                      | Collect all AI agent responses for environment | Debate Loop                | Debate Environment           | Aggregates AI replies for environment synthesis.                                                                   |
| Debate Environment       | LangChain Agent                              | Generate round summary, result, and rewritten input | Aggregate                 | JSON Output Parser 2          | Synthesizes debate round results using AI environment context.                                                     |
| Mistral Cloud Chat Model 2 | LangChain Language Model                     | Provide LLM completions for environment node   | Debate Environment         | Debate Environment           | Mistral model instance dedicated for environment processing.                                                       |
| JSON Output Parser 2     | LangChain Structured Output Parser           | Validate environment JSON output                | Debate Environment         | If No More Rounds            | Checks output schema for round summary/result/rewritten input.                                                     |
| If No More Rounds        | If                                           | Check if debate rounds completed                | JSON Output Parser 2       | End of Debate / Update Input  | Ends workflow if rounds completed; else continues debate loop.                                                     |
| Update Input             | Set                                          | Update debate variables for next round          | If No More Rounds (false)  | Guarantee Input              | Sets new input text, increments round counter, updates summaries.                                                  |
| End of Debate            | No Operation                                 | Marks workflow completion                        | If No More Rounds (true)   |                               | Final node indicating debate completion.                                                                            |
| Simple Memory            | LangChain Memory Buffer (disabled)            | Intended for AI agents' conversational memory   |                           | Debate Actor Abstraction     | Disabled in this workflow version.                                                                                   |
| Simple Memory 2          | LangChain Memory Buffer (disabled)            | Intended for environment memory                  |                           | Debate Environment           | Disabled in this workflow version.                                                                                   |
| Email Trigger (IMAP)     | Email Read IMAP (disabled)                     | Optional external trigger via email              |                           | Configure Workflow Args      | Disabled node; see above.                                                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Trigger Nodes:**
   - Add a **Schedule Trigger** node: Configure to trigger weekly on Monday, Tuesday, Wednesday, Thursday, and Friday at 12:00 PM.
   - Add a **Manual Trigger** node to allow manual invocation.

2. **Add Global Constants Node:**
   - Install and add the **Global Constants (n8n-nodes-globals)** node.
   - Configure with a credentials set containing the workflow arguments:
     - `input`: initial text string.
     - `scenario`: string describing debate scenario.
     - `rounds`: number of debate rounds (integer).
     - `ai_quantity`: number of AI agents (integer).
     - `ai_environment`: JSON object describing environment context, rewrite goals, and session ID.
     - `ai_agents`: JSON array describing each AI agent profile with properties such as name, role, nature, will, reason, likes, dislikes, writing style, and optional sessionId.
   - Connect both **Schedule Trigger** and **Manual Trigger** to this node.

3. **Prepare Initial Input:**
   - Add a **Set** node named "Prepare Input".
   - Assign variables:
     - `input` = `{{$json.constants.input}}`
     - `current_round` = 1
     - `round_summary` = "The debate begins."
     - `round_result` = "This is the first round. No last round result yet."
   - Connect **Configure Workflow Args** output to this node.

4. **Guarantee Input State:**
   - Add a **Set** node named "Guarantee Input".
   - Map the following from input or previous nodes:
     - `input`
     - `current_round`
     - `constants` (all workflow args)
     - `round_summary`
     - `round_result`
   - Connect **Prepare Input** output to this node.

5. **Set Up Round Loop:**
   - Add a **Split In Batches** node named "Round Loop".
   - Configure batch size = 1 (process one AI agent at a time).
   - Connect **Guarantee Input** output to this node.

6. **Split AI Agents:**
   - Add a **Split Out** node named "Split Out AI Agents".
   - Set the field to split: `constants.ai_agents` (array of agent profiles).
   - Connect **Round Loop** output to this node.

7. **Add Wait Control Before Agents:**
   - Add a **Wait** node, "Wait Before Sending Agents".
   - Connect **Split Out AI Agents** output to this node.

8. **Configure Debate Actor Abstraction:**
   - Add a **LangChain Agent** node.
   - Use the Mistral Cloud Chat Model (see next step) as the language model.
   - Configure prompt:
     - Input text: `"Considering your role, continue the following text.\n\"{{ $('Round Loop').item.json.input }}\"\n\nDebate Status = \"{{ $('Round Loop').item.json.round_summary }}\"\nLast Round = \"{{ $('Round Loop').item.json.round_result }}\"\n\nRemember to reply with a JSON output, as ordered!"`
     - System message template using the AI agent profile fields (`name`, `role`, `nature`, `will`, `reason`, `likes`, `dislikes`, `writing_style`) and scenario from workflow args.
   - Set output parser to expect JSON with keys: ai_name, ai_status, ai_targets, reply, cause_for_reply.
   - Set retries to 2 and 3 seconds wait on failure.
   - Connect **Wait Before Sending Agents** output to this node.

9. **Add Mistral Cloud Chat Model Node:**
   - Add **LangChain Language Model** node.
   - Select model: "mistral-small-latest".
   - Enable safe mode; max retries = 2.
   - Link credentials for Mistral Cloud API.
   - Connect this node as language model input for Debate Actor Abstraction.

10. **Add JSON Output Parser (AI Agent):**
    - Add **LangChain Structured Output Parser** node.
    - Define schema for AI agent response as per above.
    - Connect **Debate Actor Abstraction** output to this parser.

11. **Add Wait Nodes for Rate Control:**
    - Add two **Wait** nodes sequentially with 1-second delays each.
    - Connect **JSON Output Parser** -> Wait 1 -> Wait 2.

12. **Set Up Debate Loop for Round Aggregation:**
    - Add **Split In Batches** node named "Debate Loop".
    - Batch size = number of AI agents (dynamic from constants).
    - Connect **Wait 2** output to this node.

13. **Aggregate AI Agent Responses:**
    - Add **Aggregate** node.
    - Aggregate all item data into one property.
    - Connect **Debate Loop** output to Aggregate.

14. **Configure Debate Environment Node:**
    - Add **LangChain Agent** node.
    - Use second **Mistral Cloud Chat Model** node (identical settings).
    - Prompt input includes: current input text and all AI agent replies concatenated.
    - System message and rewrite goals are from `ai_environment` constants.
    - Output parser expects JSON with: round_result, round_summary, rewritten_input.
    - Set retries and wait times as before.
    - Connect **Aggregate** output to this node.

15. **Add JSON Output Parser (Environment):**
    - Add **LangChain Structured Output Parser** node.
    - Define schema to parse environment output.
    - Connect **Debate Environment** output to this node.

16. **Add Conditional Node to End or Continue:**
    - Add **If** node named "If No More Rounds".
    - Condition: Compare `current_round` with `rounds` from constants (equal means end).
    - Connect **JSON Output Parser 2** output to this node.

17. **Add Update Input Node for Next Round:**
    - Add **Set** node named "Update Input".
    - Update fields with values from environment output:
      - `input` = rewritten_input
      - `current_round` += 1
      - `round_summary`, `round_result` updated accordingly
    - Connect **If No More Rounds** (false) output to this node.

18. **Loop Back to Guarantee Input:**
    - Connect **Update Input** output to **Guarantee Input** node to restart round loop.

19. **Add End of Debate Node:**
    - Add **No Operation** node "End of Debate".
    - Connect **If No More Rounds** (true) output here.

20. **Optional: Add Disabled Nodes for Email Trigger and Memory:**
    - Add Email Trigger (IMAP) node for email-based activation (disabled by default).
    - Add Simple Memory nodes for agent and environment (disabled).

21. **Add Sticky Notes:**
    - Add descriptive sticky notes at strategic points to document the workflow structure, arguments, and usage.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                   | Context or Link                                      |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------------------|
| Workflow created by Hybroht, version 1.0. Website: https://hybroht.com                                                                                                         | Branding and author credits                          |
| Workflow uses custom node "n8n-nodes-globals" for global constants. Alternative is to use "Edit Field (Set)" nodes if unavailable.                                            | Node dependencies                                    |
| Input example given to demonstrate how the AI agents refine complex, unclear business text into clearer, professional output.                                               | Example input/output                                 |
| Workflow tested on n8n version 1.100.1, running on Linux via Podman 4.3.1.                                                                                                     | Environment specifics                                |
| The debate rounds and AI agent profiles can be fully customized via JSON workflow arguments, allowing flexible scenario simulation.                                          | Customizability                                     |
| Mistral Cloud API credentials are required for LLM nodes to function.                                                                                                         | External API requirement                             |
| Safe mode and retry settings help manage API errors and ensure robust operation.                                                                                              | Error handling and stability                         |
| Sticky notes inside workflow provide detailed explanations for each logical block, aiding maintainability and user understanding.                                            | Documentation embedded in workflow                   |
| This workflow is designed for advanced use cases involving multiple AI agents simulating debates, not for simple single-response generation.                                   | Use case specificity                                |

---

**Disclaimer:** The provided text is exclusively derived from an automated workflow created with n8n, a workflow automation tool. The processing strictly complies with content policies and contains no illegal, offensive, or protected elements. All handled data is legal and public.