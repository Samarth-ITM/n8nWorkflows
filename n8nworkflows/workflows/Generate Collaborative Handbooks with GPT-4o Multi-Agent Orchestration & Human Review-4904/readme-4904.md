Generate Collaborative Handbooks with GPT-4o Multi-Agent Orchestration & Human Review

https://n8nworkflows.xyz/workflows/generate-collaborative-handbooks-with-gpt-4o-multi-agent-orchestration---human-review-4904


# Generate Collaborative Handbooks with GPT-4o Multi-Agent Orchestration & Human Review

---

## 1. Workflow Overview

This workflow, titled **"Generate Collaborative Handbooks with GPT-4o Multi-Agent Orchestration & Human Review"**, orchestrates a sophisticated AI-driven content generation and review pipeline designed for creating and managing collaborative handbook entries within the Pyragogy AI Village ecosystem. It leverages multiple specialized GPT-4o agents working in sequence, along with human-in-the-loop approval, database persistence, and optional integration with GitHub and Slack.

**Target Use Cases:**
- Automated generation, summarization, synthesis, and review of handbook content
- Multi-agent orchestration to optimize content workflow based on input complexity and objectives
- Human review and approval loop ensuring quality and governance
- Archival and versioning of approved content in a PostgreSQL database and optional GitHub repository
- Notification via Slack upon completion

**Logical Blocks:**

- **1.1 Input Reception and Validation:** Handles incoming webhook HTTP POST requests, verifies database connectivity.
- **1.2 Meta-Orchestration:** Uses a GPT-4o Meta-Orchestrator agent to determine the optimal ordered sequence of specialized AI agents to process the input.
- **1.3 Agent Execution Loop:** Iteratively executes each AI agent in the determined sequence, preparing inputs and parsing outputs.
- **1.4 Multi-Agent Branching:** Routes execution to the appropriate AI agent node based on the current agent to run.
- **1.5 Review & Redraft Decision:** Aggregates peer review and sensemaking feedback to determine if redrafting is necessary, handling potential reprocessing loops.
- **1.6 Human-in-the-Loop Review:** Sends generated content to a human reviewer via email, awaits approval or rejection via webhook.
- **1.7 Persistence and Archival:** On human approval, stores the content and metadata in PostgreSQL, logs contributions, optionally commits to GitHub.
- **1.8 Notifications and Final Response:** Sends Slack notifications if configured, and responds to the original webhook with final outputs and contributions.

---

## 2. Block-by-Block Analysis

### 1.1 Input Reception and Validation

**Overview:**  
Receives incoming POST requests via webhook and verifies connection to the PostgreSQL database before proceeding.

**Nodes Involved:**  
- Start  
- Webhook Trigger  
- Check DB Connection

**Node Details:**

- **Start**  
  - Type: Start node (trigger)  
  - Role: Workflow entry point  
  - Config: Default, no parameters  
  - Inputs: None  
  - Outputs: Webhook Trigger  
  - Edge Cases: None

- **Webhook Trigger**  
  - Type: Webhook node  
  - Role: Receives external HTTP POST requests at path `/pyragogy/process`  
  - Config: POST method, path set to `"pyragogy/process"`, no special options  
  - Inputs: Start  
  - Outputs: Check DB Connection  
  - Edge Cases: HTTP errors, payload missing or malformed

- **Check DB Connection**  
  - Type: PostgreSQL node  
  - Role: Executes simple SQL query `SELECT 1;` to verify DB connectivity  
  - Config: Uses "Postgres Pyragogy DB" credentials  
  - Inputs: Webhook Trigger  
  - Outputs: Meta-Orchestrator  
  - Edge Cases: DB connection failure, SQL errors

---

### 1.2 Meta-Orchestration

**Overview:**  
Analyzes the input data using GPT-4o to determine the sequence of specialized AI agents best suited for processing the request.

**Nodes Involved:**  
- Meta-Orchestrator  
- Parse Orchestration Plan  
- More Agents to Run? (initial check)

**Node Details:**

- **Meta-Orchestrator**  
  - Type: OpenAI Chat node  
  - Role: Receives JSON input, returns JSON array specifying agent execution order  
  - Config: Model `gpt-4o`, response formatted as JSON object  
  - Prompt: System message defines role as Meta-orchestrator; user message includes stringified input data  
  - Credentials: OpenAI API with Pyragogy key  
  - Inputs: Check DB Connection  
  - Outputs: Parse Orchestration Plan  
  - Edge Cases: API errors, invalid JSON response, malformed input

- **Parse Orchestration Plan**  
  - Type: Function node  
  - Role: Parses the Meta-Orchestrator’s JSON output, extracts agent sequence, initializes workflow variables for looping (`agentSequence`, `currentAgentIndex`, `redraftLoopCount`)  
  - Config: JavaScript code that safely parses JSON or defaults to empty array  
  - Inputs: Meta-Orchestrator  
  - Outputs: More Agents to Run?  
  - Edge Cases: Parsing failures, missing or malformed sequences

- **More Agents to Run?** (initial iteration)  
  - Type: If node  
  - Role: Checks if there are remaining agents to run by comparing `currentAgentIndex` with sequence length  
  - Inputs: Parse Orchestration Plan  
  - Outputs: Prepare Agent Input (if true), Slack Enabled? (if false)  
  - Edge Cases: Index out of bounds, empty sequence

---

### 1.3 Agent Execution Loop

**Overview:**  
Controls iterative execution of agents, preparing input data for each, routing to specific agent nodes, and processing their outputs. Supports redraft loops based on review results.

**Nodes Involved:**  
- Prepare Agent Input  
- Route Agents with Switch  
- [Agent Nodes: Summarizer, Synthesizer, Peer Reviewer, Sensemaking, Prompt Engineer, Onboarding/Explainer, Add Handbook Metadata]  
- Process Agent Output  
- Evaluate Board Consensus  
- Check Redraft Needed  
- Handle Redraft  
- More Agents to Run? (loop continuation)

**Node Details:**

- **Prepare Agent Input**  
  - Type: Function node  
  - Role: Prepares input data for the current agent; includes any redraft feedback if applicable  
  - Inputs: More Agents to Run?  
  - Outputs: Route Agents with Switch  
  - Edge Cases: Missing previous output, improper redraft input formatting

- **Route Agents with Switch**  
  - Type: Switch node  
  - Role: Routes flow to the corresponding agent node based on `agentToRun` string  
  - Config: Matches agent names exactly among 7 known agents  
  - Inputs: Prepare Agent Input  
  - Outputs: One of the agent nodes  
  - Edge Cases: Unknown agent names, case sensitivity

- **Agent Nodes:** (All OpenAI Chat nodes except Add Handbook Metadata which is Function)  
  Each agent takes `agentInput` and applies specialized prompts:

  - **Summarizer Agent**  
    - Summarizes text into 3 key points  
    - Output: Text summary  
    - Edge Cases: Large input exceeding token limits, API errors

  - **Synthesizer Agent**  
    - Synthesizes creative new text from key points or input, optionally incorporates redraft feedback  
    - Output: Synthesized text  
    - Edge Cases: Feedback formatting, API errors

  - **Peer Reviewer Agent**  
    - Reviews text, highlights strengths/weaknesses, gives actionable suggestions, outputs JSON with `major_issue` flag  
    - Output: JSON object  
    - Edge Cases: JSON parsing failures, incomplete feedback

  - **Sensemaking Agent**  
    - Analyzes input in context of DB data, identifies patterns/gaps, suggests directions, outputs JSON with `major_issue` flag  
    - Output: JSON object  
    - Edge Cases: Missing DB context, JSON parsing

  - **Prompt Engineer Agent**  
    - Refines or generates optimal prompt for next agent based on context, outputs JSON with `major_issue` flag  
    - Output: JSON object  
    - Edge Cases: JSON parsing

  - **Onboarding/Explainer Agent**  
    - Explains current process or provides guidance based on input  
    - Output: Text  
    - Edge Cases: Large input, API errors

  - **Add Handbook Metadata**  
    - Prepares metadata (title, tags, phase, rhythm) for handbook content  
    - Inputs: From previous agent output  
    - Output: JSON enriched with metadata  
    - Edge Cases: Missing fields defaulted

- **Process Agent Output**  
  - Type: Function node  
  - Role: Extracts and parses agent output, logs contribution, increments currentAgentIndex unless redrafting is triggered  
  - Handles special Archivist output states (approved, rejected, pending)  
  - Inputs: Agent nodes or Human Review merge  
  - Outputs: More Agents to Run?  
  - Edge Cases: Missing or malformed output, JSON parse errors, redraft loops, Archivist human feedback handling

- **Evaluate Board Consensus**  
  - Type: Function node  
  - Role: Aggregates `major_issue` flags from Peer Reviewer, Sensemaking, Prompt Engineer agents to decide if redrafting needed (majority threshold = 2)  
  - Collects redraft feedback text for next iteration  
  - Inputs: Prompt Engineer Agent output  
  - Outputs: Check Redraft Needed  
  - Edge Cases: Missing or invalid JSON, partial agent outputs

- **Check Redraft Needed**  
  - Type: If node  
  - Role: Decides to enter redraft loop if redraft is needed and loop count < 2  
  - Inputs: Evaluate Board Consensus  
  - Outputs: Handle Redraft (true), Process Agent Output (false)  
  - Edge Cases: Loop count limits prevents infinite loops

- **Handle Redraft**  
  - Type: Function node  
  - Role: Increments redraft loop counter, resets `currentAgentIndex` to Synthesizer index for reprocessing, attaches redraft feedback to input  
  - Inputs: Check Redraft Needed  
  - Outputs: More Agents to Run?  
  - Edge Cases: Synthesizer not in sequence, edge cases with index resets

---

### 1.4 Human-in-the-Loop Review & Archival

**Overview:**  
After final agent output (typically Archivist or Add Handbook Metadata), the content is prepared for human review via email. The workflow waits for human approval or rejection, then persists or logs accordingly. Optionally commits to GitHub and sends Slack notifications.

**Nodes Involved:**  
- Generate Content for Review  
- Generate Review ID  
- Send Review Request Email  
- Wait for Human Approval  
- Human Decision Split  
- Save to handbook_entries  
- Prepare Approved Contribution Data  
- Save Agent Contribution (Approved)  
- Generate GitHub File Path  
- GitHub Enabled?  
- Commit to GitHub (Approved)  
- Log Human Rejection  
- Merge Archivist Paths  
- Notify Slack  
- Slack Enabled?  
- Final Response

**Node Details:**

- **Generate Content for Review**  
  - Type: Function node  
  - Role: Builds Markdown content with YAML front matter including title, tags, phase, rhythm  
  - Inputs: Add Handbook Metadata output  
  - Outputs: Generate Review ID  
  - Edge Cases: Proper escaping of quotes in YAML, missing metadata

- **Generate Review ID**  
  - Type: Function node  
  - Role: Generates unique UUID to track review instance  
  - Inputs: Generate Content for Review  
  - Outputs: Send Review Request Email  
  - Edge Cases: UUID generation failure unlikely

- **Send Review Request Email**  
  - Type: Email Send node  
  - Role: Sends email to configured human reviewer with proposed content and approval/rejection links embedding reviewId  
  - Inputs: Generate Review ID  
  - Outputs: Wait for Human Approval  
  - Credentials: Configured email credentials (SMTP or similar)  
  - Edge Cases: Email delivery failure, incorrect links

- **Wait for Human Approval**  
  - Type: Wait node (webhook mode)  
  - Role: Pauses workflow awaiting webhook callback with reviewId and status (approved/rejected)  
  - Timeout: 1 hour  
  - Inputs: Send Review Request Email  
  - Outputs: Human Decision Split  
  - Edge Cases: Timeout without response, webhook failures

- **Human Decision Split**  
  - Type: If node  
  - Role: Branches based on human decision status (`approved` or other)  
  - Inputs: Wait for Human Approval  
  - Outputs: Save to handbook_entries (approved), Log Human Rejection (rejected)  
  - Edge Cases: Unexpected status values

- **Save to handbook_entries**  
  - Type: PostgreSQL node  
  - Role: Inserts approved handbook content and metadata into `handbook_entries` table  
  - Inputs: Human Decision Split (approved)  
  - Credentials: Postgres Pyragogy DB  
  - Edge Cases: DB insert errors, data validation

- **Prepare Approved Contribution Data**  
  - Type: Function node  
  - Role: Formats contribution log details for the approval event, including metadata and status  
  - Inputs: Save to handbook_entries  
  - Outputs: Save Agent Contribution (Approved)  
  - Edge Cases: Missing fields fallback

- **Save Agent Contribution (Approved)**  
  - Type: PostgreSQL node  
  - Role: Inserts contribution record into `agent_contributions` table for auditing  
  - Inputs: Prepare Approved Contribution Data  
  - Credentials: Postgres Pyragogy DB  
  - Outputs: Generate GitHub File Path  
  - Edge Cases: DB errors

- **Generate GitHub File Path**  
  - Type: Function node  
  - Role: Constructs GitHub file path for versioned handbook content using slugified title and timestamp  
  - Inputs: Save Agent Contribution (Approved)  
  - Outputs: GitHub Enabled?  
  - Edge Cases: Special characters in title, time formatting

- **GitHub Enabled?**  
  - Type: If node  
  - Role: Checks if environment variable `GITHUB_ACCESS_TOKEN` is present to proceed with GitHub commit  
  - Inputs: Generate GitHub File Path  
  - Outputs: Commit to GitHub (Approved) or Merge Archivist Paths  
  - Edge Cases: Missing token disables GitHub integration

- **Commit to GitHub (Approved)**  
  - Type: GitHub node  
  - Role: Creates or updates handbook content file in configured GitHub repo with commit message referencing approval  
  - Credentials: GitHub API with access token  
  - Inputs: GitHub Enabled?  
  - Outputs: Merge Archivist Paths  
  - Edge Cases: API errors, repo permission issues

- **Log Human Rejection**  
  - Type: Function node  
  - Role: Logs rejection event details including comments and proposed content to console, prepares rejection output  
  - Inputs: Human Decision Split (rejected)  
  - Outputs: Merge Archivist Paths  
  - Edge Cases: Missing rejection comments

- **Merge Archivist Paths**  
  - Type: Merge node  
  - Role: Merges outputs from approval and rejection branches on `reviewId` to unify flow  
  - Inputs: Commit to GitHub (Approved), Log Human Rejection  
  - Outputs: Process Agent Output  
  - Edge Cases: Failures merging data sets

- **Notify Slack**  
  - Type: Slack node  
  - Role: Sends notification to Slack webhook with summary of input, final output, and executed agents  
  - Inputs: More Agents to Run? (false branch via Slack Enabled?)  
  - Config: Uses environment variable `SLACK_WEBHOOK_URL`  
  - Edge Cases: Slack webhook failures

- **Slack Enabled?**  
  - Type: If node  
  - Role: Checks if Slack webhook URL environment variable is set  
  - Inputs: More Agents to Run?  
  - Outputs: Notify Slack or Final Response

- **Final Response**  
  - Type: Respond to Webhook node  
  - Role: Sends final JSON response back to original webhook caller containing final output, contributions, and agent sequence  
  - Inputs: Notify Slack or Slack Enabled?  
  - Edge Cases: Response formatting errors

---

## 3. Summary Table

| Node Name                 | Node Type           | Functional Role                                       | Input Node(s)             | Output Node(s)                      | Sticky Note                                      |
|---------------------------|---------------------|------------------------------------------------------|---------------------------|-----------------------------------|-------------------------------------------------|
| Start                     | Start               | Entry point                                          | None                      | Webhook Trigger                   |                                                 |
| Webhook Trigger           | Webhook             | Receives HTTP POST input                             | Start                     | Check DB Connection               |                                                 |
| Check DB Connection       | Postgres            | Verifies DB connection                               | Webhook Trigger           | Meta-Orchestrator                 |                                                 |
| Meta-Orchestrator         | OpenAI Chat         | Determines agent execution sequence                  | Check DB Connection       | Parse Orchestration Plan          |                                                 |
| Parse Orchestration Plan  | Function            | Parses agent sequence, initializes workflow vars     | Meta-Orchestrator         | More Agents to Run?               |                                                 |
| More Agents to Run?       | If                  | Checks if more agents remain to execute              | Parse Orchestration Plan  | Prepare Agent Input, Slack Enabled? |                                                 |
| Prepare Agent Input       | Function            | Prepares input for current agent                      | More Agents to Run?        | Route Agents with Switch          |                                                 |
| Route Agents with Switch  | Switch              | Routes to agent node based on current agent          | Prepare Agent Input       | Multiple agent nodes              |                                                 |
| Summarizer Agent          | OpenAI Chat         | Summarizes text to 3 key points                       | Route Agents with Switch  | Process Agent Output              |                                                 |
| Synthesizer Agent         | OpenAI Chat         | Synthesizes creative text, incorporates feedback     | Route Agents with Switch  | Process Agent Output              |                                                 |
| Peer Reviewer Agent       | OpenAI Chat         | Reviews text, outputs JSON with major_issue flag     | Route Agents with Switch  | Sensemaking Agent                |                                                 |
| Sensemaking Agent         | OpenAI Chat         | Analyzes input with context, outputs JSON with flags | Peer Reviewer Agent       | Prompt Engineer Agent             |                                                 |
| Prompt Engineer Agent     | OpenAI Chat         | Generates/refines prompt for next agent               | Sensemaking Agent         | Evaluate Board Consensus          |                                                 |
| Onboarding/Explainer Agent| OpenAI Chat         | Explains process or guidance                           | Route Agents with Switch  | Process Agent Output              |                                                 |
| Add Handbook Metadata     | Function            | Adds metadata (title, tags, etc.)                     | Route Agents with Switch  | Generate Content for Review       |                                                 |
| Process Agent Output      | Function            | Parses output, logs contribution, increments index   | Agent nodes, Merge Archivist Paths | More Agents to Run?           |                                                 |
| Evaluate Board Consensus  | Function            | Aggregates review flags, decides redraft necessity   | Prompt Engineer Agent     | Check Redraft Needed              |                                                 |
| Check Redraft Needed      | If                  | Branches to redraft or continue                        | Evaluate Board Consensus  | Handle Redraft, Process Agent Output |                                                 |
| Handle Redraft            | Function            | Manages redraft loop, resets index to Synthesizer    | Check Redraft Needed      | More Agents to Run?               |                                                 |
| Generate Content for Review| Function           | Prepares content with YAML front matter for review   | Add Handbook Metadata     | Generate Review ID                |                                                 |
| Generate Review ID        | Function            | Creates UUID for review tracking                       | Generate Content for Review | Send Review Request Email        |                                                 |
| Send Review Request Email | Email Send          | Sends content review email to human reviewer          | Generate Review ID        | Wait for Human Approval          |                                                 |
| Wait for Human Approval   | Wait (webhook mode) | Pauses until human review response                     | Send Review Request Email | Human Decision Split             |                                                 |
| Human Decision Split      | If                  | Branches based on human approval or rejection         | Wait for Human Approval   | Save to handbook_entries, Log Human Rejection |                                                 |
| Save to handbook_entries  | Postgres            | Saves approved content into DB                          | Human Decision Split      | Prepare Approved Contribution Data |                                                 |
| Prepare Approved Contribution Data| Function      | Prepares contribution log data                         | Save to handbook_entries  | Save Agent Contribution (Approved) |                                                 |
| Save Agent Contribution (Approved)| Postgres      | Logs agent contribution post approval                  | Prepare Approved Contribution Data | Generate GitHub File Path      |                                                 |
| Generate GitHub File Path | Function            | Creates versioned GitHub file path                      | Save Agent Contribution (Approved) | GitHub Enabled?                |                                                 |
| GitHub Enabled?           | If                  | Checks for GitHub token to enable commit               | Generate GitHub File Path | Commit to GitHub (Approved), Merge Archivist Paths |                                                 |
| Commit to GitHub (Approved)| GitHub             | Commits approved content file to GitHub                | GitHub Enabled?           | Merge Archivist Paths            |                                                 |
| Log Human Rejection       | Function            | Logs rejection details, prepares rejection output      | Human Decision Split      | Merge Archivist Paths            |                                                 |
| Merge Archivist Paths     | Merge               | Merges approval and rejection paths                    | Commit to GitHub (Approved), Log Human Rejection | Process Agent Output |                                                 |
| Slack Enabled?            | If                  | Checks Slack webhook URL presence                       | More Agents to Run?       | Notify Slack, Final Response     |                                                 |
| Notify Slack              | Slack               | Sends workflow completion notification to Slack        | Slack Enabled?            | Final Response                  |                                                 |
| Final Response            | Respond to Webhook  | Sends final JSON response to the original webhook caller | Notify Slack, Slack Enabled? | None                          |                                                 |

---

## 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n and add a Start node.**

2. **Add a Webhook node:**
   - Name: `Webhook Trigger`
   - HTTP Method: POST
   - Path: `pyragogy/process`
   - Connect Start → Webhook Trigger

3. **Add a PostgreSQL node:**
   - Name: `Check DB Connection`
   - Operation: Execute Query
   - Query: `SELECT 1; -- Verifica connessione DB`
   - Credentials: Configure with your Pyragogy Postgres DB credentials
   - Connect Webhook Trigger → Check DB Connection

4. **Add OpenAI Chat node:**
   - Name: `Meta-Orchestrator`
   - Model: `gpt-4o`
   - Authentication: OpenAI API key credential (Pyragogy OpenAI)
   - Response Format: JSON Object
   - Messages:
     - System: "You are the Meta-orchestrator... [as per prompt]"
     - User: `Input Data:\n{{ JSON.stringify($json.body) }}`
   - Connect Check DB Connection → Meta-Orchestrator

5. **Add Function node:**
   - Name: `Parse Orchestration Plan`
   - Code: JavaScript to parse JSON array from Meta-Orchestrator output, initialize workflow variables `agentSequence`, `currentAgentIndex`, and `redraftLoopCount` to 0.
   - Pass initial input and first agent to run.
   - Connect Meta-Orchestrator → Parse Orchestration Plan

6. **Add If node:**
   - Name: `More Agents to Run?`
   - Condition: boolean expression `{{$workflow.currentAgentIndex < $workflow.agentSequence.length}} == true`
   - Connect Parse Orchestration Plan → More Agents to Run?  
   - True branch → Prepare Agent Input  
   - False branch → Slack Enabled?

7. **Add Function node:**
   - Name: `Prepare Agent Input`
   - Prepare input for current agent including redraft feedback if any.
   - Connect More Agents to Run? (True) → Prepare Agent Input

8. **Add Switch node:**
   - Name: `Route Agents with Switch`
   - Mode: JSON  
   - Value: `{{$json.agentToRun}}`  
   - Conditions: String equals "Summarizer", "Synthesizer", "Peer Reviewer", "Sensemaking Agent", "Prompt Engineer", "Onboarding/Explainer", "Archivist"
   - Connect Prepare Agent Input → Route Agents with Switch

9. **Add OpenAI Chat nodes for each agent:**

   For each agent, configure as follows (connect respective switch output to agent node):

   - **Summarizer Agent**  
     - Model: gpt-4o  
     - System prompt: "You are the Summarizer Agent..."  
     - User prompt: "Text to summarize:\n{{ $json.agentInput }}"  

   - **Synthesizer Agent**  
     - Model: gpt-4o  
     - System prompt: "You are the Synthesizer Agent..."   
     - User prompt: "Input for synthesis:\n{{ $json.agentInput }}\n\n{{ $json.redraftFeedback ? 'Feedback...' : '' }}"  

   - **Peer Reviewer Agent**  
     - Model: gpt-4o  
     - Response format: JSON object  
     - System prompt: "You are the Peer Reviewer Agent..."  
     - User prompt: "Text to review:\n{{ $json.agentInput }}"  

   - **Sensemaking Agent**  
     - Model: gpt-4o  
     - Response format: JSON object  
     - System prompt: "You are the Sensemaking Agent..."  
     - User prompt: "Input to analyze:\n{{ $json.agentInput }}\n\nContext from DB (if available):\n{{ $json.dbContext }}"  

   - **Prompt Engineer Agent**  
     - Model: gpt-4o  
     - Response format: JSON object  
     - System prompt: "You are the Prompt Engineer Agent..."  
     - User prompt: "Current context:\n{{ JSON.stringify($json) }}\nNext agent: {{ $workflow.agentSequence[$workflow.currentAgentIndex + 1] || 'None' }}"  

   - **Onboarding/Explainer Agent**  
     - Model: gpt-4o  
     - System prompt: "You are the Onboarding/Explainer Agent..."  
     - User prompt: "Explain the following:\n{{ $json.agentInput }}"  

10. **Add Function node:**
    - Name: `Add Handbook Metadata`
    - Extract or default metadata fields: title, tags, phase, rhythm from input  
    - Connect Route Agents with Switch → Add Handbook Metadata (on Archivist branch)

11. **Add Function node:**
    - Name: `Process Agent Output`
    - Extract output from agent node, parse JSON if needed, increment currentAgentIndex (unless redrafting), log contribution  
    - Handle special Archivist output states  
    - Connect all agent nodes → Process Agent Output

12. **Add Function node:**
    - Name: `Evaluate Board Consensus`
    - Parse outputs from Peer Reviewer, Sensemaking, Prompt Engineer; count `major_issue` flags; decide if redraft needed (>= 2)  
    - Connect Prompt Engineer Agent → Evaluate Board Consensus

13. **Add If node:**
    - Name: `Check Redraft Needed`
    - Condition: `{{$json.redraftNeeded && $workflow.redraftLoopCount < 2}} == true`
    - Connect Evaluate Board Consensus → Check Redraft Needed  
    - True → Handle Redraft  
    - False → Process Agent Output

14. **Add Function node:**
    - Name: `Handle Redraft`
    - Increment redraftLoopCount, reset currentAgentIndex to Synthesizer index, attach redraft feedback  
    - Connect Check Redraft Needed (True) → Handle Redraft  
    - Handle Redraft → More Agents to Run?

15. **Add Function node:**
    - Name: `Generate Content for Review`
    - Compose YAML front matter and final markdown content for review email  
    - Connect Add Handbook Metadata → Generate Content for Review

16. **Add Function node:**
    - Name: `Generate Review ID`
    - Generate UUID to track review  
    - Connect Generate Content for Review → Generate Review ID

17. **Add Email Send node:**
    - Name: `Send Review Request Email`
    - Configure SMTP credentials  
    - To: human-reviewer@example.com  
    - Subject and Body include content and approval/rejection webhook links with `reviewId`  
    - Connect Generate Review ID → Send Review Request Email

18. **Add Wait node:**
    - Name: `Wait for Human Approval`
    - Mode: Webhook  
    - Webhook Path: `pyragogy/review-feedback`  
    - Timeout: 1 hour  
    - Match Field: reviewId  
    - Connect Send Review Request Email → Wait for Human Approval

19. **Add If node:**
    - Name: `Human Decision Split`
    - Condition: `{{$json.query.status === 'approved'}} == true`  
    - Connect Wait for Human Approval → Human Decision Split  
    - True → Save to handbook_entries  
    - False → Log Human Rejection

20. **Add PostgreSQL node:**
    - Name: `Save to handbook_entries`
    - Insert new row with columns: title, content, version, created_by, tags, phase, rhythm  
    - Credentials: Pyragogy Postgres DB  
    - Connect Human Decision Split (approved) → Save to handbook_entries

21. **Add Function node:**
    - Name: `Prepare Approved Contribution Data`
    - Format contribution log data with entryId and metadata  
    - Connect Save to handbook_entries → Prepare Approved Contribution Data

22. **Add PostgreSQL node:**
    - Name: `Save Agent Contribution (Approved)`
    - Insert row into `agent_contributions` table  
    - Connect Prepare Approved Contribution Data → Save Agent Contribution (Approved)

23. **Add Function node:**
    - Name: `Generate GitHub File Path`
    - Create versioned filename with slug and timestamp  
    - Connect Save Agent Contribution (Approved) → Generate GitHub File Path

24. **Add If node:**
    - Name: `GitHub Enabled?`
    - Condition: check if environment variable `GITHUB_ACCESS_TOKEN` exists  
    - Connect Generate GitHub File Path → GitHub Enabled?  
    - True → Commit to GitHub (Approved)  
    - False → Merge Archivist Paths

25. **Add GitHub node:**
    - Name: `Commit to GitHub (Approved)`
    - Operation: create/update file in repository  
    - Use access token credential  
    - Connect GitHub Enabled? (True) → Commit to GitHub (Approved)

26. **Add Function node:**
    - Name: `Log Human Rejection`
    - Logs rejection info and passes rejection output downstream  
    - Connect Human Decision Split (rejected) → Log Human Rejection

27. **Add Merge node:**
    - Name: `Merge Archivist Paths`
    - Merge on property `reviewId` from Commit to GitHub and Log Human Rejection  
    - Connect Commit to GitHub (Approved), Log Human Rejection → Merge Archivist Paths

28. **Connect Merge Archivist Paths → Process Agent Output** to continue or finalize loop.

29. **Add If node:**
    - Name: `Slack Enabled?`
    - Condition: check if environment variable `SLACK_WEBHOOK_URL` exists  
    - Connect More Agents to Run? (False branch) → Slack Enabled?  
    - True → Notify Slack  
    - False → Final Response

30. **Add Slack node:**
    - Name: `Notify Slack`
    - Post message with input, final output, agents run  
    - Connect Slack Enabled? (True) → Notify Slack

31. **Add Respond to Webhook node:**
    - Name: `Final Response`
    - Respond with JSON containing finalOutput, contributions, agentSequence  
    - Connect Notify Slack → Final Response  
    - Connect Slack Enabled? (False) → Final Response

---

## 5. General Notes & Resources

| Note Content                                                                                              | Context or Link                                            |
|----------------------------------------------------------------------------------------------------------|------------------------------------------------------------|
| Workflow leverages GPT-4o model for advanced multi-agent orchestration and content generation.           | OpenAI GPT-4o model documentation                           |
| Human review is integrated via email with actionable approve/reject webhooks for governance.             | Email configuration and webhook handling in n8n            |
| PostgreSQL database schema requires tables: `handbook_entries` and `agent_contributions` with expected columns. | DB schema design for Pyragogy AI Village                    |
| GitHub integration depends on environment variables for repository owner/name and token.                  | GitHub API docs and n8n GitHub node configuration           |
| Slack notifications are optional, triggered by presence of `SLACK_WEBHOOK_URL` environment variable.      | Slack incoming webhook setup                                |
| Workflow includes redraft loop logic with a maximum of 2 iterations to avoid infinite cycles.             | Redraft control logic in function nodes                      |
| Email review links must be updated with actual n8n public URL replacing `your_n8n_url` in the email node. | Email templates require customization                        |

---

**Disclaimer:**  
The provided text is derived exclusively from an automated workflow created with n8n, a workflow automation tool. This processing strictly adheres to applicable content policies and contains no illegal, offensive, or protected material. All manipulated data is legal and public.

---