Generate Complete Database Schemas with Claude Multi-Agent System for SQL Databases

https://n8nworkflows.xyz/workflows/generate-complete-database-schemas-with-claude-multi-agent-system-for-sql-databases-9544


# Generate Complete Database Schemas with Claude Multi-Agent System for SQL Databases

### 1. Workflow Overview

This workflow automates the generation of complete, production-ready database schemas for SQL databases using a multi-agent AI system powered by Claude (Anthropic). It is designed for users who provide detailed database requirements via a form. The system iteratively creates, reviews, optimizes, and finally generates executable SQL scripts for the specified database technology, supporting PostgreSQL, MySQL, MSSQL, and MariaDB.

**Target Use Cases:**  
- Rapid prototyping of scalable, normalized database schemas  
- AI-assisted database design and review for startups and enterprises  
- Automated SQL migration script generation with best practices  
- Iterative improvement and quality assurance via AI feedback loops

**Logical Blocks:**  
- **1.1 Input Reception and Initialization:** Receives user input via form and initializes workflow variables.  
- **1.2 Multi-Agent Schema Design Pipeline:** Runs through three AI agents sequentially—Architect, Reviewer, and Optimizer—to create and refine the schema iteratively.  
- **1.3 Quality Gate and Iteration Control:** Evaluates the optimizer's score and decides to proceed or retry up to 3 times.  
- **1.4 SQL Script Generation:** Converts final schema design into an executable SQL script.  
- **1.5 Execution and Response Handling:** Optionally executes the SQL script in PostgreSQL and returns success or error responses.  
- **1.6 User Interaction and Completion:** Displays final results or errors back to user via form completion node.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception and Initialization

- **Overview:** Collects database requirements from the user via a web form and initializes key variables for iteration and feedback.  
- **Nodes Involved:** Form Submission, Initialize Variables

**Node Details:**

- **Form Submission**  
  - Type: Form Trigger  
  - Role: Collect user input on database requirements including technology, industry, scale, workflows, integrations, special requirements, and contact email.  
  - Configuration: Various form fields including textarea, dropdown, number, and email with required validation.  
  - Outputs: JSON object with user responses.  
  - Edge cases: Missing required fields; malformed input; slow user response.

- **Initialize Variables**  
  - Type: Set node  
  - Role: Initialize `iteration_count` to 0, `previous_feedback` to "=" (empty placeholder), and store form data for workflow context.  
  - Configuration: Sets three variables from JSON input.  
  - Inputs: Output from Form Submission.  
  - Outputs: Initialized context for agents.  
  - Edge cases: None significant; fails if input missing.

---

#### 1.2 Multi-Agent Schema Design Pipeline

- **Overview:** Runs three AI agents in sequence, each powered by Claude Sonnet 4.5, to create, review, and optimize the proposed database schema using the input data and iterative feedback.  
- **Nodes Involved:** Architect Model, Agent 1: Database Architect, Reviewer Model, Agent 2: Schema Reviewer, Optimizer Model, Agent 3: Optimizer & Scorer, Track Iteration

**Node Details:**

- **Architect Model**  
  - Type: Anthropic LM Chat node  
  - Role: Base model configuration for Agent 1.  
  - Parameters: Temperature 0.3, topP 0.9 for balanced creativity and consistency.  
  - Credentials: Anthropic API  
  - Output: Model response to Agent 1.

- **Agent 1: Database Architect**  
  - Type: Langchain Agent node  
  - Role: Designs initial database schema as JSON based on form data and previous feedback if any.  
  - Prompt: Detailed instructions to design scalable, normalized DB schema with tables, columns, PK, indexes, FKs, enums, extensions, relationships, reasoning.  
  - Input: Form data and previous feedback from Initialize Variables or Track Iteration.  
  - Output: JSON design of schema.  
  - Edge cases: Incomplete feedback; model hallucinations; overly complex schemas.

- **Reviewer Model**  
  - Type: Anthropic LM Chat node  
  - Role: Base model configuration for Agent 2.  
  - Parameters: Temperature 0.1, topP 0.85 for strict analytical review.  
  - Credentials: Anthropic API

- **Agent 2: Schema Reviewer**  
  - Type: Langchain Agent node  
  - Role: Reviews Agent 1 schema for normalization, indexing, data types, constraints, performance, scalability, security; outputs structured feedback with severity levels.  
  - Prompt includes original requirements and schema from Agent 1.  
  - Output: Structured feedback with CRITICAL to LOW issues and positives.  
  - Edge cases: Missed issues; false positives; incomplete feedback.

- **Optimizer Model**  
  - Type: Anthropic LM Chat node  
  - Role: Base model config for Agent 3.  
  - Parameters: Temperature 0.5, topP 0.95 for balanced polish and creativity.  
  - Credentials: Anthropic API

- **Agent 3: Optimizer & Scorer**  
  - Type: Langchain Agent node  
  - Role: Applies final polish, adds advanced features (materialized views, helper functions), monitoring recommendations, comprehensive score card with grades and justifications, implementation guidance.  
  - Prompt includes original requirements, schema, and review feedback.  
  - Output: Score card and enhanced schema.  
  - Edge cases: Over-optimism; missing critical fixes.

- **Track Iteration**  
  - Type: Set node  
  - Role: Increment iteration count, update previous feedback with Agent 3 output for retry loop, preserve form data.  
  - Inputs: Agent 3 output and Initialize Variables data.  
  - Outputs: Updated context for next iteration or decision.

---

#### 1.3 Quality Gate and Iteration Control

- **Overview:** Evaluates if the schema score is good enough (grade A or B) to proceed to SQL generation or if the workflow should retry up to 3 times for improvement.  
- **Nodes Involved:** Is Score A or B?, Can Retry?, Max Iterations Reached

**Node Details:**

- **Is Score A or B?**  
  - Type: If node  
  - Role: Regex match on Agent 3's output to detect overall grade A or B.  
  - True path: Proceed to SQL generation.  
  - False path: Proceed to retry or max iteration check.

- **Can Retry?**  
  - Type: If node  
  - Role: Checks if current iteration count ≤ 3 to allow retry.  
  - True path: Loop back to Agent 1 for redesign with previous feedback.  
  - False path: Proceed to Max Iterations Reached.

- **Max Iterations Reached**  
  - Type: Set node  
  - Role: Sets message indicating max 3 iterations reached and proceeds with best design available.  
  - Outputs: Passes through current iteration and form data for final steps.

---

#### 1.4 SQL Script Generation

- **Overview:** Converts the finalized database schema design into a detailed, production-ready SQL migration script, tailored for the specified database technology.  
- **Nodes Involved:** SQL Generator Model, Agent 4: Generate SQL Script

**Node Details:**

- **SQL Generator Model**  
  - Type: Anthropic LM Chat node  
  - Role: Base model config for Agent 4.  
  - Parameters: Temperature 0.2, topP 0.85 for precision.  
  - Credentials: Anthropic API

- **Agent 4: Generate SQL Script**  
  - Type: Langchain Agent node  
  - Role: Generates full SQL migration script with transaction handling, extension creation (PostgreSQL), enums, tables, indexes, constraints, comments, idempotency, rollback script.  
  - Input: Database technology, final schema, and optimizations.  
  - Output: Plain SQL script text (no markdown or commentary).  
  - Edge cases: Syntax errors; dialect incompatibility; large output truncation.

---

#### 1.5 Execution and Response Handling

- **Overview:** Optionally executes the generated SQL script on a PostgreSQL database and handles success or failure responses to provide user feedback.  
- **Nodes Involved:** If User Is Using Excel (condition node), Execute SQL in PostgreSQL, Success Response, Error Response

**Node Details:**

- **If User Is Using Excel**  
  - Type: If node  
  - Role: Checks if the database technology contains "postgres" (case insensitive) to determine if auto-execution is possible.  
  - True path: Execute SQL in PostgreSQL  
  - False path: Bypass execution and show form completion only.

- **Execute SQL in PostgreSQL**  
  - Type: PostgreSQL node  
  - Role: Executes the SQL script generated by Agent 4 against configured PostgreSQL database.  
  - Configuration: Uses stored PostgreSQL credentials.  
  - On error: Continues workflow with error output.  
  - Output: Success or error JSON.

- **Success Response**  
  - Type: Set node  
  - Role: Constructs a JSON response with status "success," message, SQL script, schema design, review feedback, score card, iteration count, and database tech for final output.  
  - Input: Execute SQL success output and agent outputs.

- **Error Response**  
  - Type: Set node  
  - Role: Constructs a JSON response with status "failed," error message, SQL script, schema, feedback, score card, iteration count, and tech. Advises manual use of script if execution fails.  
  - Input: Execute SQL error output and agent outputs.

---

#### 1.6 User Interaction and Completion

- **Overview:** Presents the final workflow output to the user via a form completion node, either after successful execution or in failure cases without execution.  
- **Nodes Involved:** Form, Form End Without Execute

**Node Details:**

- **Form**  
  - Type: Form node  
  - Role: Completion interface that shows final success or error JSON back to user with a title and message.  
  - Input: From Success Response or Error Response nodes.

- **Form End Without Execute**  
  - Type: Form node  
  - Role: Completion interface used when SQL script is generated but not executed (non-PostgreSQL DBs).  
  - Input: From If User Is Using Excel false branch.

---

### 3. Summary Table

| Node Name               | Node Type                          | Functional Role                                   | Input Node(s)              | Output Node(s)                 | Sticky Note                                                |
|-------------------------|----------------------------------|-------------------------------------------------|----------------------------|-------------------------------|------------------------------------------------------------|
| Form Submission         | Form Trigger                     | Collects user database requirements              |                            | Initialize Variables           | ## 📝 Form Intake - user fills database requirements       |
| Initialize Variables    | Set                              | Initializes iteration and feedback variables      | Form Submission            | Agent 1: Database Architect    | ## 🔧 Setup Phase - prepares data for agent pipeline       |
| Architect Model         | Anthropic LM Chat                | Base model config for Architect Agent             |                            | Agent 1: Database Architect    | ## 🤖 Agent 1: Architect - initial schema design           |
| Agent 1: Database Architect | Langchain Agent               | Designs initial database schema                    | Initialize Variables       | Agent 2: Schema Reviewer       | ## 🤖 Agent 1: Architect - initial schema design           |
| Reviewer Model          | Anthropic LM Chat                | Base model config for Reviewer Agent               |                            | Agent 2: Schema Reviewer       | ## 🤖 Agent 2: Reviewer - quality control & validation     |
| Agent 2: Schema Reviewer | Langchain Agent                 | Reviews schema for normalization & best practices | Agent 1: Database Architect | Agent 3: Optimizer & Scorer    | ## 🤖 Agent 2: Reviewer - quality control & validation     |
| Optimizer Model         | Anthropic LM Chat                | Base model config for Optimizer Agent              |                            | Agent 3: Optimizer & Scorer    | ## 🤖 Agent 3: Optimizer - final polish & scoring          |
| Agent 3: Optimizer & Scorer | Langchain Agent              | Refines schema, adds features, scores design      | Agent 2: Schema Reviewer   | Is Score A or B?               | ## 🤖 Agent 3: Optimizer - final polish & scoring          |
| Track Iteration         | Set                              | Tracks iteration count and feedback for retry     | Agent 3: Optimizer & Scorer | Can Retry?                    | ## 🔁 Retry Loop - iterative improvement system            |
| Is Score A or B?        | If                               | Checks if score grade is A or B                     | Agent 3: Optimizer & Scorer | Agent 4: Generate SQL Script / Track Iteration | ## 🔄 Quality Gate - ensures high-quality output            |
| Can Retry?              | If                               | Checks if iteration count ≤ 3 for retries          | Track Iteration            | Agent 1: Database Architect / Max Iterations Reached | ## 🔄 Quality Gate - ensures high-quality output            |
| Max Iterations Reached  | Set                              | Sets message when max retries reached              | Can Retry?                 | Agent 4: Generate SQL Script   | ## 🔄 Quality Gate - ensures high-quality output            |
| SQL Generator Model     | Anthropic LM Chat                | Base model config for SQL script generation        |                            | Agent 4: Generate SQL Script   | ## 🤖 Agent 4: SQL Generator - converts schema to SQL      |
| Agent 4: Generate SQL Script | Langchain Agent             | Generates production-ready SQL migration script    | Max Iterations Reached / Is Score A or B? | If User Is Using Excel       | ## 🤖 Agent 4: SQL Generator - converts schema to SQL      |
| If User Is Using Excel  | If                               | Checks if DB tech is PostgreSQL to decide execution | Agent 4: Generate SQL Script | Execute SQL in PostgreSQL / Form End Without Execute | ## Query Execute - auto runs script on PostgreSQL          |
| Execute SQL in PostgreSQL | PostgreSQL                     | Executes SQL script on PostgreSQL database         | If User Is Using Excel     | Success Response / Error Response | ## Query Execute - auto runs script on PostgreSQL          |
| Success Response        | Set                              | Constructs success response JSON                    | Execute SQL in PostgreSQL  | Form                         | ## ✅ Execution Validation - handles success gracefully    |
| Error Response          | Set                              | Constructs failure response JSON                    | Execute SQL in PostgreSQL  | Form                         | ## ✅ Execution Validation - handles failure gracefully    |
| Form                    | Form                             | Shows final success or failure message to user     | Success Response / Error Response |                           | ## ✅ Execution Validation - handles success & failures    |
| Form End Without Execute | Form                            | Shows final message when script not executed       | If User Is Using Excel (false) |                           | ## ✅ Execution Validation - handles success & failures    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Submission Node**  
   - Type: Form Trigger  
   - Configure fields:  
     - What are you building? (textarea, required)  
     - Database Technology (dropdown: PostgreSQL, MySQL, MSSQL, MariaDB, required)  
     - Industry/Niche (text, required)  
     - What data do you need to track? (textarea)  
     - Team Size (number)  
     - Expected Scale (text)  
     - Key Workflows (textarea)  
     - Any integrations needed? (text)  
     - Special Requirements (textarea)  
     - Your Email (email, required)  
   - Set form title and description accordingly.

2. **Add Set Node "Initialize Variables"**  
   - Assign:  
     - iteration_count = 0 (number)  
     - previous_feedback = "=" (string)  
     - form_data = entire JSON from Form Submission (object)  
   - Connect output of Form Submission to this node.

3. **Add Anthropic LM Chat Node "Architect Model"**  
   - Model: Claude Sonnet 4.5  
   - Temperature: 0.3, topP: 0.9  
   - Connect to Agent 1 (see next step).

4. **Add Langchain Agent Node "Agent 1: Database Architect"**  
   - Text input: Template including all form_data fields and previous_feedback/iteration_count.  
   - System message: Senior DB architect instructions for schema design output as structured JSON.  
   - Connect input to Initialize Variables output.  
   - Connect AI model to Architect Model node.

5. **Add Anthropic LM Chat Node "Reviewer Model"**  
   - Model: Claude Sonnet 4.5  
   - Temperature: 0.1, topP: 0.85  
   - Connect to Agent 2 node.

6. **Add Langchain Agent Node "Agent 2: Schema Reviewer"**  
   - Input: JSON stringified form_data and Agent 1 output schema.  
   - System message: Reviewer checklist, output structured feedback with severity levels.  
   - Connect input from Agent 1 output.  
   - Connect AI model to Reviewer Model node.

7. **Add Anthropic LM Chat Node "Optimizer Model"**  
   - Model: Claude Sonnet 4.5  
   - Temperature: 0.5, topP: 0.95  
   - Connect to Agent 3 node.

8. **Add Langchain Agent Node "Agent 3: Optimizer & Scorer"**  
   - Input: Original requirements, proposed schema, review feedback.  
   - System message: Instructions to polish, add advanced features, score, provide guidance.  
   - Connect input from Agent 2 output.  
   - Connect AI model to Optimizer Model node.

9. **Add Set Node "Track Iteration"**  
   - Assign:  
     - iteration_count = previous iteration_count + 1  
     - previous_feedback = Agent 3 output  
     - form_data = initial form_data  
   - Connect input from Agent 3 output.

10. **Add If Node "Can Retry?"**  
    - Condition: iteration_count ≤ 3  
    - True: loop back to Agent 1 (for redesign with updated feedback)  
    - False: proceed to Max Iterations Reached node.

11. **Add Set Node "Max Iterations Reached"**  
    - Assign message indicating max attempts reached.  
    - Pass through iteration_count and form_data.  
    - Connect input from Can Retry? false output.

12. **Add If Node "Is Score A or B?"**  
    - Regex check on Agent 3 output for grades A or B.  
    - True: proceed to SQL generation.  
    - False: proceed to Track Iteration (retry loop).

13. **Add Anthropic LM Chat Node "SQL Generator Model"**  
    - Model: Claude Sonnet 4.5  
    - Temperature: 0.2, topP: 0.85  
    - Connect to Agent 4 node.

14. **Add Langchain Agent Node "Agent 4: Generate SQL Script"**  
    - Input: Database technology, final schema, optimizations.  
    - System message: Detailed instructions for generating production-ready SQL migration script with transaction handling, comments, rollback, idempotency.  
    - Connect AI model to SQL Generator Model node.

15. **Add If Node "If User Is Using Excel"**  
    - Condition: Check if database technology string contains "postgres" (case-insensitive).  
    - True: Execute SQL node.  
    - False: Skip execution and proceed to form completion.

16. **Add PostgreSQL Node "Execute SQL in PostgreSQL"**  
    - Credentials: PostgreSQL account credentials.  
    - Query: SQL script output from Agent 4.  
    - Operation: executeQuery  
    - On error: Continue error output.

17. **Add Set Node "Success Response"**  
    - On successful SQL execution, set JSON response with status "success", message, SQL script, schema design, review feedback, score card, iterations, and database tech.  
    - Connect from Execute SQL success output.

18. **Add Set Node "Error Response"**  
    - On failure, set JSON response with status "failed", error message, SQL script, schema, review feedback, score card, iterations, and tech.  
    - Connect from Execute SQL error output.

19. **Add Form Node "Form"**  
    - Operation: completion  
    - Completion title: "SQL Creation Finished"  
    - Completion message: bind to JSON of Success Response or Error Response nodes.  
    - Connect from Success Response and Error Response nodes.

20. **Add Form Node "Form End Without Execute"**  
    - Operation: completion  
    - Completion title: "SQL Creation Finished"  
    - Completion message: bind to JSON from Agent 4 output (when SQL is not executed).  
    - Connect from If User Is Using Excel false output.

21. **Add Sticky Notes**  
    - Add descriptive sticky notes at each major section for user guidance and customization tips.

---

### 5. General Notes & Resources

| Note Content                                                                                                                          | Context or Link                                                                                  |
|----------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Average workflow execution time is 60-90 seconds with ~85% success on first attempt. Cost per run estimated at $0.15-$0.30.           | Workflow Overview Sticky Note                                                                    |
| Suggested improvements include A/B testing agent prompts, caching patterns, pre-built industry templates, and parallel execution.     | Workflow Overview Sticky Note                                                                    |
| Retry loop currently maxes at 3 iterations; increasing retries or adding manual review can improve quality but increase cost.          | Retry Loop Sticky Note                                                                           |
| Agent prompts are carefully tuned with temperature and topP to balance creativity, strictness, and polish across the design pipeline.| Agent Sticky Notes 1-4                                                                            |
| SQL generated includes transaction handling, idempotency, and rollback script for safe deployment.                                    | Agent 4 Sticky Note                                                                              |
| Execution node supports automatic running of scripts only for PostgreSQL; other DBs require manual execution of generated SQL.        | Query Execute Sticky Note                                                                        |
| Error handling ensures user receives usable output even if execution fails, including the generated SQL script for manual use.         | Execution Validation Sticky Note                                                                 |

---

**Disclaimer:** The content above is derived exclusively from an automated n8n workflow utilizing AI language models. All data processed is lawful and publicly provided. No illegal or protected content is included.