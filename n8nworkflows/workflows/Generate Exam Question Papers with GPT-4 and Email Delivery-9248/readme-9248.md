Generate Exam Question Papers with GPT-4 and Email Delivery

https://n8nworkflows.xyz/workflows/generate-exam-question-papers-with-gpt-4-and-email-delivery-9248


# Generate Exam Question Papers with GPT-4 and Email Delivery

### 1. Workflow Overview

This workflow automates the generation of exam question papers based on user-submitted syllabus content using the GPT-4 AI model and delivers the formatted question paper via email. It is designed for educators or institutions who want to quickly create structured question papers for different syllabus units and send them to specified recipients.

The workflow is logically divided into these blocks:

- **1.1 Input Reception:** Collects syllabus details and recipient email via a web form.
- **1.2 AI Question Generation:** Uses three AI agents to generate questions for Parts A, B, and C of the exam paper, each with specific Bloom’s Taxonomy levels and mark allocations.
- **1.3 Output Parsing and Merging:** Parses the AI-generated JSON outputs, merges them into a single structured object.
- **1.4 Formatting Question Paper:** Converts the merged questions into a styled HTML question paper template.
- **1.5 Email Delivery:** Sends the generated question paper via Gmail to the user’s email with BCC to an internal address.
- **1.6 Sticky Notes:** Provide contextual instructions and tips for users and maintainers.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Reception

**Overview:**  
This block captures user input for subject code/name, syllabus content for two units, and the email address to which the generated question paper will be sent.

**Nodes Involved:**  
- On form submission  
- Sticky Note2  

**Node Details:**

- **On form submission**  
  - Type: Form Trigger (Webhook)  
  - Role: Entry point; collects form data with fields: subject name/code, syllabus for unit 1, syllabus for unit 2, recipient email (required).  
  - Configuration: Form titled "Syllabus Submission" with appropriate fields including textarea and email.  
  - Input: HTTP POST from form submission  
  - Output: JSON object containing user inputs for downstream AI processing.  
  - Failure Modes: Missing required email, malformed input, webhook connectivity issues.  
  - Sticky Note2 reminds users to input subject code/name, syllabus, and email.

- **Sticky Note2**  
  - Content: "In the form, input the subject code/name, syllabus, and email to send the question paper"  
  - Role: User instruction on input format.

---

#### 2.2 AI Question Generation

**Overview:**  
This block uses three separate AI agents (LangChain agents with GPT-4 models) to generate exam questions for Parts A, B, and C of the question paper, each adhering to specific instructions and Bloom’s Taxonomy cognitive levels.

**Nodes Involved:**  
- OpenAI Chat Model (3 instances: base models for each agent)  
- Part A QP Agent  
- Part B QP Agent1  
- Part C QP Agent  
- Structured Output Parser (3 instances, one per agent)  
- Sticky Note3, Sticky Note4, Sticky Note5  

**Node Details:**

- **OpenAI Chat Model / OpenAI Chat Model1 / OpenAI Chat Model2**  
  - Type: LangChain OpenAI Chat Model  
  - Role: Provide GPT-4 powered language model backend to AI agents.  
  - Configuration: Models used are "gpt-4-turbo-2024-04-09" or "gpt-4-turbo" or "gpt-4o-mini" for different agents.  
  - Credential: OpenAI API key linked.  
  - Inputs: Prompt text from respective AI Agent nodes.  
  - Outputs: Raw chat completions.  
  - Edge Cases: API auth failure, rate limits, model unavailability, timeout, malformed prompts.  

- **Part A QP Agent**  
  - Type: LangChain Agent  
  - Role: Generate 4 questions (2 marks each) from syllabus units 1 & 2, focusing on Bloom’s levels Remember and Understand.  
  - Configuration: Prompt instructs to create exactly 4 questions without numbering, outputting JSON with "QuestionsA" array.  
  - Input: JSON from form submission, prompt text uses syllabus fields via expressions.  
  - Output: Parsed structured JSON.  
  - Error Handling: Continues on error output.  
  - Sticky Note3: "This AI Agent will generate four 2 mark questions."

- **Part B QP Agent1**  
  - Type: LangChain Agent  
  - Role: Generate 4 questions (13 marks each) from syllabus units 1 & 2, Bloom’s levels Apply and Analyse.  
  - Configuration: Prompt instructs to create 4 questions with choices, output JSON with "QuestionsB".  
  - Input: Form data via expressions.  
  - Output: Parsed structured JSON.  
  - Error Handling: Continues on regular output.  
  - Sticky Note4: "This AI Agent will generate four 13 mark questions with each question having 2 choices."

- **Part C QP Agent**  
  - Type: LangChain Agent  
  - Role: Generate 2 questions (14 marks each) from syllabus unit 1, Bloom’s levels Analyze and Evaluate.  
  - Configuration: Prompt instructs 2 questions output as JSON array "QuestionsC".  
  - Input: Form data for syllabus unit 1.  
  - Output: Parsed structured JSON.  
  - Sticky Note5: "This AI Agent will generate two 14 mark questions with each question having 2 choices."

- **Structured Output Parser / Structured Output Parser1 / Structured Output Parser2**  
  - Type: LangChain Structured Output Parser  
  - Role: Validate and parse AI raw text outputs into JSON objects according to schema examples.  
  - Configured with JSON schema examples matching each AI agent’s output format.  
  - Input: AI agent output.  
  - Output: Structured JSON for merging.  
  - Edge Cases: Parsing failure if AI output deviates from expected JSON format.

---

#### 2.3 Output Parsing and Merging

**Overview:**  
This block merges the outputs from the three AI agents into a single consolidated JSON object representing all parts of the question paper.

**Nodes Involved:**  
- Merge  
- Merge1  
- Code  

**Node Details:**

- **Merge**  
  - Type: Merge (mode: default)  
  - Role: Merge results from Part A and Part B AI agents into one stream.  
  - Input: Outputs from Part A QP Agent and Part B QP Agent1 nodes.  
  - Output: Combined data for further merging.  
  - Error Handling: Continues on regular output.

- **Merge1**  
  - Type: Merge  
  - Role: Merge output of Merge node with Part C QP Agent output.  
  - Input: Combined Part A+B and Part C outputs.  
  - Output: Single stream with all three parts of questions.

- **Code**  
  - Type: Code node (JavaScript)  
  - Role: Consolidate the three question arrays into one JSON object with keys QuestionsA, QuestionsB, QuestionsC.  
  - Code snippet extracts question arrays from the merged items and returns a single JSON object.  
  - Input: Merged outputs.  
  - Output: Unified question paper JSON object for formatting.

---

#### 2.4 Formatting Question Paper

**Overview:**  
Transforms the merged question data into a nicely styled HTML document representing the final exam question paper.

**Nodes Involved:**  
- QP Formatter with HTML  
- Sticky Note (for template guidance)  

**Node Details:**

- **QP Formatter with HTML**  
  - Type: HTML node  
  - Role: Applies a predefined HTML template with inline CSS styling to present the question paper.  
  - Configuration:  
    - Includes header with college info, dynamic subject code/name from form data.  
    - Sections for Part A, B, and C questions with numbering and marks breakdown.  
    - Uses n8n expressions to inject question arrays into the HTML.  
    - Footer includes generation date and credit.  
  - Input: JSON question paper object from Code node and form data.  
  - Output: HTML content as string for email body.  
  - Sticky Note: "In the HTML section enter the html code of the question paper template and call the generated question from AI Agents."

---

#### 2.5 Email Delivery

**Overview:**  
Sends the formatted question paper to the email address provided by the user, with a BCC copy to an internal address.

**Nodes Involved:**  
- Gmail  
- Sticky Note1  

**Node Details:**

- **Gmail**  
  - Type: Gmail node (OAuth2)  
  - Role: Sends email with subject and message body containing the formatted question paper HTML.  
  - Configuration:  
    - Recipient email dynamically sourced from form input.  
    - Subject line uses the subject name/code from form.  
    - Message body contains the HTML output of the question paper.  
    - BCC to internal email "weki9631@gmail.com".  
    - Execute once per workflow run.  
  - Credential: Gmail OAuth2 account linked.  
  - Input: HTML from QP Formatter node, recipient email from form.  
  - Output: Email send status.  
  - Edge Cases: Authentication errors, email delivery failure, invalid recipient address.  
  - Sticky Note1: "You will receive the generated question paper in the email."

---

### 3. Summary Table

| Node Name           | Node Type                           | Functional Role                             | Input Node(s)                         | Output Node(s)               | Sticky Note                                                                                  |
|---------------------|-----------------------------------|--------------------------------------------|-------------------------------------|-----------------------------|----------------------------------------------------------------------------------------------|
| On form submission   | Form Trigger                      | Collect syllabus and recipient email       | n/a (trigger)                       | Part A QP Agent, Part B QP Agent1, Part C QP Agent | In the form, input the subject code/name, syllabus, and email to send the question paper      |
| OpenAI Chat Model    | LangChain OpenAI Chat Model       | Provide GPT-4 model backend for Part A     | Part A QP Agent                     | Part A QP Agent              |                                                                                              |
| OpenAI Chat Model1   | LangChain OpenAI Chat Model       | Provide GPT-4 model backend for Part B     | Part B QP Agent1                   | Part B QP Agent1             |                                                                                              |
| OpenAI Chat Model2   | LangChain OpenAI Chat Model       | Provide GPT-4 model backend for Part C     | Part C QP Agent                    | Part C QP Agent              |                                                                                              |
| Part A QP Agent      | LangChain Agent                   | Generate 4 questions (2 marks each) Part A | On form submission, OpenAI Chat Model | Merge                       | This AI Agent will generate four 2 mark questions                                           |
| Part B QP Agent1     | LangChain Agent                   | Generate 4 questions (13 marks each) Part B | On form submission, OpenAI Chat Model1 | Merge                       | This AI Agent will generate four 13 mark questions with each question having 2 choices       |
| Part C QP Agent      | LangChain Agent                   | Generate 2 questions (14 marks each) Part C | On form submission, OpenAI Chat Model2 | Merge1                      | This AI Agent will generate two 14 mark questions with each question having 2 choices        |
| Structured Output Parser | LangChain Output Parser Structured | Parse Part A AI output into JSON           | Part A QP Agent                    | Part A QP Agent              |                                                                                              |
| Structured Output Parser1| LangChain Output Parser Structured | Parse Part B AI output into JSON           | Part B QP Agent1                  | Part B QP Agent1             |                                                                                              |
| Structured Output Parser2| LangChain Output Parser Structured | Parse Part C AI output into JSON           | Part C QP Agent                   | Part C QP Agent              |                                                                                              |
| Merge               | Merge                            | Combine Part A and Part B question outputs | Part A QP Agent, Part B QP Agent1 | Merge1                      |                                                                                              |
| Merge1              | Merge                            | Combine previous merge with Part C output  | Merge, Part C QP Agent             | Code                        |                                                                                              |
| Code                | Code (JavaScript)                | Consolidate all question arrays into one   | Merge1                           | QP Formatter with HTML       |                                                                                              |
| QP Formatter with HTML| HTML                            | Format full question paper into styled HTML| Code                            | Gmail                       | In the HTML section enter the html code of the question paper template and call the generated question from AI Agents |
| Gmail               | Gmail (OAuth2)                   | Send formatted question paper by email     | QP Formatter with HTML            | n/a                         | You will receive the generated question paper in the email.                                 |
| Sticky Note          | Sticky Note                     | Instruction on HTML template usage          | n/a                               | n/a                         | In the HTML section enter the html code of the question paper template and call the generated question from AI Agents |
| Sticky Note1         | Sticky Note                     | Instruction on email reception               | n/a                               | n/a                         | You will receive the generated question paper in the email.                                 |
| Sticky Note2         | Sticky Note                     | Instruction on form input                    | n/a                               | n/a                         | In the form, input the subject code/name, syllabus, and email to send the question paper      |
| Sticky Note3         | Sticky Note                     | Describes Part A AI Agent question output   | n/a                               | n/a                         | This AI Agent will generate four 2 mark questions                                           |
| Sticky Note4         | Sticky Note                     | Describes Part B AI Agent question output   | n/a                               | n/a                         | This AI Agent will generate four 13 mark questions with each question having 2 choices       |
| Sticky Note5         | Sticky Note                     | Describes Part C AI Agent question output   | n/a                               | n/a                         | This AI Agent will generate two 14 mark questions with each question having 2 choices        |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger Node ("On form submission"):**  
   - Type: Form Trigger  
   - Configure form titled "Syllabus Submission" with fields:  
     - "Name of the subject with code" (text)  
     - "syllabus for unit 1" (textarea)  
     - "syllabus for unit 2" (text)  
     - "Enter your email to send the Exam Question Paper." (email, required)  

2. **Create three OpenAI Chat Model nodes:**  
   - Node 1 ("OpenAI Chat Model"): Model "gpt-4-turbo-2024-04-09" for Part A agent.  
   - Node 2 ("OpenAI Chat Model1"): Model "gpt-4-turbo" for Part B agent.  
   - Node 3 ("OpenAI Chat Model2"): Model "gpt-4o-mini" for Part C agent.  
   - Set credentials: OpenAI API key.

3. **Create AI Agent nodes:**  
   - "Part A QP Agent":  
     - Type: LangChain Agent  
     - Prompt instructs generation of 4 questions (2 marks) from syllabus units 1 & 2, Bloom’s Remember and Understand.  
     - Use expressions to inject syllabus data from form fields.  
     - Link input from "On form submission" and "OpenAI Chat Model".  
     - Enable output parser with JSON schema matching 4 questions array.  
     - On error: continue with error output.

   - "Part B QP Agent1":  
     - Similar setup, prompt for 4 questions (13 marks) Bloom’s Apply and Analyse from both syllabus units.  
     - Link input from "On form submission" and "OpenAI Chat Model1".  
     - Output parser enabled with proper JSON schema.  
     - On error: continue with regular output.

   - "Part C QP Agent":  
     - Prompt for 2 questions (14 marks) Bloom’s Analyze and Evaluate from syllabus unit 1 only.  
     - Link input from "On form submission" and "OpenAI Chat Model2".  
     - Output parser with JSON schema for 2 questions array.

4. **Create Structured Output Parsers:**  
   - For each AI Agent node’s output, create a Structured Output Parser node with the corresponding JSON schema example.  
   - Connect AI Agent output to its parser node.

5. **Merge Outputs:**  
   - Create a "Merge" node to combine outputs from Part A and Part B parsers.  
   - Create a second "Merge1" node to merge the previous merge output with Part C parser output.

6. **Create a Code Node ("Code"):**  
   - Write JavaScript to consolidate all question arrays into a single JSON object with keys QuestionsA, QuestionsB, QuestionsC.  
   - Input from "Merge1".  
   - Output to formatting node.

7. **Create an HTML Node ("QP Formatter with HTML"):**  
   - Paste the provided HTML template that formats the question paper.  
   - Use n8n expressions to inject subject name/code and question arrays dynamically.  
   - Input from "Code" node.

8. **Create Gmail Node ("Gmail"):**  
   - Configure sender credentials (OAuth2 Gmail account).  
   - Set recipient dynamically from form email input.  
   - Set subject line to the subject name/code from form input.  
   - Set message body to the HTML from formatting node.  
   - Add BCC email "weki9631@gmail.com".  
   - Execute once per run.

9. **Add Sticky Notes:**  
   - Add notes near form input node explaining required fields.  
   - Add notes near AI agents describing question types and marks.  
   - Add notes near HTML formatter with instructions on editing the template.  
   - Add notes near Gmail node about email receipt.

10. **Connect Nodes:**  
    - "On form submission" → All three AI Agents.  
    - Each AI Agent → respective OpenAI Chat Model → AI Agent.  
    - Each AI Agent → respective Structured Output Parser.  
    - Part A & Part B parsers → Merge → Merge1 (with Part C parser).  
    - Merge1 → Code → QP Formatter with HTML → Gmail.  

11. **Credentials Setup:**  
    - Configure OpenAI API key credentials and assign to OpenAI Chat Model nodes.  
    - Configure Gmail OAuth2 credentials and assign to Gmail node.

12. **Test and Validate:**  
    - Submit form with valid inputs.  
    - Check AI-generated JSON outputs for correctness.  
    - Ensure email is received with the formatted question paper.  
    - Troubleshoot API or formatting errors as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                 | Context or Link                                        |
|----------------------------------------------------------------------------------------------|-------------------------------------------------------|
| The HTML question paper template uses inline CSS for clean formatting and dynamic content.  | Located in "QP Formatter with HTML" node.             |
| Generated question paper includes credit to "Gracewell89 via N8N" and generation date.      | Footer section of the HTML template.                   |
| Workflow tags include "email delivery", "exam", "questions", and "question paper".          | Workflow metadata for categorization.                  |
| Gmail node includes BCC to "weki9631@gmail.com" for internal record keeping.                | Gmail node configuration.                              |
| Uses GPT-4 based models with specific Bloom’s Taxonomy levels to create pedagogically sound questions. | AI agent prompt design.                                |

---

**Disclaimer:** The workflow was created solely with n8n automation and complies fully with content policies. No illegal or protected content is included. All data processed is lawful and public.