Avoid Asking Redundant Questions with Dynamically Generated Forms using OpenAI 

https://n8nworkflows.xyz/workflows/avoid-asking-redundant-questions-with-dynamically-generated-forms-using-openai--3062


# Avoid Asking Redundant Questions with Dynamically Generated Forms using OpenAI 

### 1. Workflow Overview

This workflow, titled **"Avoid Asking Redundant Questions with Dynamically Generated Forms using OpenAI"**, is designed to optimize form-based data collection by dynamically generating follow-up questions based on user input. It targets scenarios where capturing comprehensive information is critical but user experience must be preserved by avoiding redundant or already answered questions. A typical use case is an AI consultancy capturing leads, where an initial open-ended response is analyzed to determine which specific predefined questions remain unanswered.

The workflow is logically divided into four main blocks:

- **1.1 Initial Form Pages:** Collects basic user information and an open-ended business overview.
- **1.2 AI Response Analysis:** Uses an LLM (OpenAI GPT-4o-mini) with chain-of-thought reasoning and a structured output parser to analyze the open-ended response and identify which specific questions have been answered.
- **1.3 Clean Up Analysis:** Filters out questions already answered and prepares the remaining questions for dynamic form generation.
- **1.4 Generate Final Form Page & Completion:** Presents the dynamically generated form with only unanswered questions, then shows a completion page after submission.

---

### 2. Block-by-Block Analysis

#### 2.1 Initial Form Pages

- **Overview:**  
  This block gathers the user's basic contact information and an open-ended description of their current situation and interest in AI automation. It serves as the data input foundation for subsequent AI analysis.

- **Nodes Involved:**  
  - Get Basic Information  
  - Get Business Overview

- **Node Details:**

  - **Get Basic Information**  
    - *Type:* Form Trigger  
    - *Role:* Entry point collecting user’s name, company, job title, and email.  
    - *Configuration:* Form titled "Get in Touch" with four required fields: Name, Company Name, Job Title, Email (email type).  
    - *Input/Output:* Triggered by webhook; outputs collected form data.  
    - *Edge Cases:* Missing required fields will prevent submission; webhook connectivity issues may block form access.

  - **Get Business Overview**  
    - *Type:* Form  
    - *Role:* Collects a detailed, open-ended text response describing the user's current situation and AI automation interest.  
    - *Configuration:* Single required textarea field labeled "Please describe your current situation and why you are interested in automating with AI".  
    - *Input/Output:* Triggered after basic info; outputs the long text response.  
    - *Edge Cases:* Empty or very short responses may reduce AI analysis quality.

---

#### 2.2 AI Response Analysis

- **Overview:**  
  This block analyzes the open-ended business overview and basic info using an LLM to determine which specific predefined questions have been answered. It uses chain-of-thought reasoning and outputs a structured JSON indicating answered status per question.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Analyse Response  
  - Structured Output Parser

- **Node Details:**

  - **OpenAI Chat Model**  
    - *Type:* LangChain OpenAI Chat Model  
    - *Role:* Executes the GPT-4o-mini model to process the prompt.  
    - *Configuration:* Model set to "gpt-4o-mini" with default options; uses OpenAI API credentials.  
    - *Input/Output:* Receives prompt text from "Analyse Response"; outputs raw LLM response.  
    - *Version:* 1.2  
    - *Edge Cases:* API quota limits, network errors, or invalid credentials may cause failures.  
    - *Modification Note:* Can be replaced with a different language model if needed.

  - **Analyse Response**  
    - *Type:* LangChain Chain LLM  
    - *Role:* Constructs a detailed prompt combining user job title and open-ended response, instructing the LLM to analyze answers to five critical questions with chain-of-thought reasoning.  
    - *Configuration:* Prompt includes customer info and five specific questions to check for answers. Output format is structured and parsed.  
    - *Input/Output:* Takes form data inputs; outputs structured JSON via the parser.  
    - *Version:* 1.5  
    - *Edge Cases:* Prompt misconfiguration or unexpected input formats may cause parsing errors.  
    - *Modification Note:* Prompt can be customized to fit different use cases.

  - **Structured Output Parser**  
    - *Type:* LangChain Structured Output Parser  
    - *Role:* Parses the LLM output into a JSON structure with fields: question, has_been_answered (boolean), and reasoning.  
    - *Configuration:* Uses a JSON schema example specifying expected output format.  
    - *Input/Output:* Parses raw LLM output for downstream filtering.  
    - *Version:* 1.2  
    - *Edge Cases:* Parsing failures if LLM output deviates from schema.

---

#### 2.3 Clean Up Analysis

- **Overview:**  
  This block filters out questions already answered according to the AI analysis and prepares the remaining questions for dynamic form generation.

- **Nodes Involved:**  
  - Split Out Analysis  
  - Remove Already Answered Questions  
  - Prepare For Form Generation  
  - Aggregate For Form Generation

- **Node Details:**

  - **Split Out Analysis**  
    - *Type:* Split Out  
    - *Role:* Splits the array of question analysis objects into individual items for filtering.  
    - *Configuration:* Splits on the field `output.response`.  
    - *Input/Output:* Receives structured JSON array; outputs individual question objects.  
    - *Version:* 1  
    - *Edge Cases:* Empty or malformed arrays may cause no output.

  - **Remove Already Answered Questions**  
    - *Type:* Filter  
    - *Role:* Filters out questions where `has_been_answered` is true, keeping only unanswered questions.  
    - *Configuration:* Condition checks `has_been_answered == false`.  
    - *Input/Output:* Receives individual question objects; outputs only unanswered ones.  
    - *Version:* 2.2  
    - *Edge Cases:* If all questions are answered, output will be empty, resulting in no follow-up questions.

  - **Prepare For Form Generation**  
    - *Type:* Set  
    - *Role:* Transforms each unanswered question into a form field definition.  
    - *Configuration:* Sets `fieldLabel` to the question text, `requiredField` to true, and `fieldType` to "textarea".  
    - *Input/Output:* Receives filtered questions; outputs form field objects.  
    - *Version:* 3.4  
    - *Edge Cases:* Empty input leads to no form fields generated.

  - **Aggregate For Form Generation**  
    - *Type:* Aggregate  
    - *Role:* Aggregates all form field objects into a single array for the dynamic form node.  
    - *Configuration:* Aggregates all item data.  
    - *Input/Output:* Receives multiple form field items; outputs a single aggregated array.  
    - *Version:* 1  
    - *Edge Cases:* No items to aggregate results in empty form fields.

---

#### 2.4 Generate Final Form Page & Completion

- **Overview:**  
  This block dynamically generates the final form page with only unanswered questions and then displays a form completion message after submission.

- **Nodes Involved:**  
  - Clarification Questions  
  - End Form

- **Node Details:**

  - **Clarification Questions**  
    - *Type:* Form  
    - *Role:* Presents the dynamically generated form fields (unanswered questions) for user input.  
    - *Configuration:* Form fields are defined dynamically via JSON from aggregated form field data (`jsonOutput` set to `={{ $json.data }}`).  
    - *Input/Output:* Receives form field definitions; outputs user responses.  
    - *Webhook:* Has a webhook ID for form submission.  
    - *Version:* 1  
    - *Edge Cases:* If no questions remain, form may be empty or skipped.

  - **End Form**  
    - *Type:* Form Completion  
    - *Role:* Displays a completion message confirming form submission.  
    - *Configuration:* Title "Form Completed" and message "Thank you for answering these questions. We'll be in touch soon!"  
    - *Input/Output:* Triggered after final form submission; no further output.  
    - *Version:* 1  
    - *Edge Cases:* None significant; serves as user feedback.

---

### 3. Summary Table

| Node Name                  | Node Type                                | Functional Role                               | Input Node(s)           | Output Node(s)              | Sticky Note                                                                                                  |
|----------------------------|-----------------------------------------|-----------------------------------------------|-------------------------|-----------------------------|--------------------------------------------------------------------------------------------------------------|
| Get Basic Information       | Form Trigger                            | Collects basic user info                       | —                       | Get Business Overview        | ## 1. Initial Form Pages                                                                                      |
| Get Business Overview       | Form                                    | Collects open-ended business overview         | Get Basic Information    | Analyse Response             | ## 1. Initial Form Pages                                                                                      |
| OpenAI Chat Model           | LangChain OpenAI Chat Model             | Runs GPT-4o-mini LLM                           | Analyse Response         | Analyse Response             | ### Modification: Replace this sub-node to use a different language model                                    |
| Analyse Response            | LangChain Chain LLM                     | Constructs prompt and analyzes user response  | Get Business Overview    | Split Out Analysis           | ### Modification: Modify the prompt to suit your use case                                                    |
| Structured Output Parser    | LangChain Structured Output Parser      | Parses LLM output into structured JSON        | OpenAI Chat Model        | Analyse Response             |                                                                                                              |
| Split Out Analysis          | Split Out                              | Splits array of question analysis objects     | Analyse Response         | Remove Already Answered Questions | ## 3. Clean Up Analysis                                                                                      |
| Remove Already Answered Questions | Filter                              | Filters out answered questions                 | Split Out Analysis       | Prepare For Form Generation  | ## 3. Clean Up Analysis                                                                                      |
| Prepare For Form Generation | Set                                    | Prepares form field definitions                | Remove Already Answered Questions | Aggregate For Form Generation | ## 3. Clean Up Analysis                                                                                      |
| Aggregate For Form Generation | Aggregate                            | Aggregates form fields into array              | Prepare For Form Generation | Clarification Questions      | ## 4. Generate Final Form Page & End Form                                                                    |
| Clarification Questions     | Form                                   | Presents dynamic form with unanswered questions | Aggregate For Form Generation | End Form                    | ## 4. Generate Final Form Page & End Form                                                                    |
| End Form                   | Form Completion                        | Shows form completion message                  | Clarification Questions  | —                           | ## 4. Generate Final Form Page & End Form                                                                    |
| Sticky Note4               | Sticky Note                            | Workflow overview and detailed description     | —                       | —                           | # Avoid Asking Redundant Questions with Dynamically Generated Forms using OpenAI ... (full content)          |
| Sticky Note5               | Sticky Note                            | Setup instructions                             | —                       | —                           | ## Setup: 1. Add your OpenAI credentials 2. Test Get Basic Information node 3. Complete form 4. Modify prompt  |
| Sticky Note                | Sticky Note                            | Label for initial form pages block             | —                       | —                           | ## 1. Initial Form Pages                                                                                      |
| Sticky Note1               | Sticky Note                            | Label for AI response analysis block           | —                       | —                           | ## 2. Analyse Response                                                                                        |
| Sticky Note2               | Sticky Note                            | Label for clean up analysis block               | —                       | —                           | ## 3. Clean Up Analysis                                                                                       |
| Sticky Note3               | Sticky Note                            | Label for final form generation and completion | —                       | —                           | ## 4. Generate Final Form Page & End Form                                                                    |
| Sticky Note6               | Sticky Note                            | Prompt modification reminder                    | —                       | —                           | ### Modification: Modify the prompt to suit your use case                                                   |
| Sticky Note8               | Sticky Note                            | Model replacement reminder                       | —                       | —                           | ### Modification: Replace this sub-node to use a different language model                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Get Basic Information" node**  
   - Type: Form Trigger  
   - Configure webhook (auto-generated)  
   - Form Title: "Get in Touch"  
   - Add required fields:  
     - Name (text)  
     - Company Name (text)  
     - Job Title (text)  
     - Email (email)  
   - Save node.

2. **Create "Get Business Overview" node**  
   - Type: Form  
   - Configure webhook (auto-generated)  
   - Add one required field:  
     - Field Type: Textarea  
     - Label: "Please describe your current situation and why you are interested in automating with AI"  
   - Connect output of "Get Basic Information" to this node.

3. **Create "OpenAI Chat Model" node**  
   - Type: LangChain OpenAI Chat Model  
   - Select model: "gpt-4o-mini"  
   - Add OpenAI API credentials (ensure valid API key)  
   - No additional options needed.

4. **Create "Structured Output Parser" node**  
   - Type: LangChain Structured Output Parser  
   - Paste JSON schema example specifying output structure with fields: question (string), has_been_answered (boolean), reasoning (string).  
   - Connect output of "OpenAI Chat Model" to this node’s input parser.

5. **Create "Analyse Response" node**  
   - Type: LangChain Chain LLM  
   - Configure prompt as follows (use expression syntax to inject form data):  
     ```
     ## Analysis Task

     Analyze the following customer response to the question "Please describe your current situation and why you are interested in automating with AI."

     Customer Information:
     - Job Title: {{ $('Get Basic Information').item.json['Job Title'] }}
     - Response: {{ $json['Please describe your current situation and why you are interested in automating with AI'] }}

     ## Required Information
     Identify whether the customer's response clearly addresses each of these critical questions:

     1. What specific goals are you looking to achieve with automation?
     2. Does the company have any existing automation workflows already in place?
     3. Is the respondent a decision-maker in the business? (This can be inferred from their job title if it indicates a leadership position such as CEO, Founder, Director, etc.)
     4. Which specific business functions or departments are you looking to automate? (Examples: Sales, Marketing, HR, Finance, Customer Service, Supply Chain, etc.)
     5. What does your current IT infrastructure look like?

     ## Output Format
     Analyse each question with whether you believe that the question has already been answered. Go step by step and use reasoning.
     ```
   - Enable output parser and select the "Structured Output Parser" node.  
   - Connect "Get Business Overview" output to this node.  
   - Connect this node’s output to "OpenAI Chat Model" input.

6. **Create "Split Out Analysis" node**  
   - Type: Split Out  
   - Field to split out: `output.response`  
   - Connect output of "Analyse Response" to this node.

7. **Create "Remove Already Answered Questions" node**  
   - Type: Filter  
   - Condition: `has_been_answered == false` (boolean false)  
   - Connect output of "Split Out Analysis" to this node.

8. **Create "Prepare For Form Generation" node**  
   - Type: Set  
   - Assign fields:  
     - `fieldLabel` = `{{ $json.question }}`  
     - `requiredField` = `true`  
     - `fieldType` = `"textarea"`  
   - Connect output of "Remove Already Answered Questions" to this node.

9. **Create "Aggregate For Form Generation" node**  
   - Type: Aggregate  
   - Aggregate all item data  
   - Connect output of "Prepare For Form Generation" to this node.

10. **Create "Clarification Questions" node**  
    - Type: Form  
    - Configure webhook (auto-generated)  
    - Set form definition mode to "JSON"  
    - Set JSON output to `={{ $json.data }}` (dynamic form fields from aggregation)  
    - Connect output of "Aggregate For Form Generation" to this node.

11. **Create "End Form" node**  
    - Type: Form Completion  
    - Set completion title: "Form Completed"  
    - Set completion message: "Thank you for answering these questions. We'll be in touch soon!"  
    - Connect output of "Clarification Questions" to this node.

12. **Connect nodes in order:**  
    - Get Basic Information → Get Business Overview → Analyse Response → Split Out Analysis → Remove Already Answered Questions → Prepare For Form Generation → Aggregate For Form Generation → Clarification Questions → End Form

13. **Credential Setup:**  
    - Add OpenAI API credentials with valid API key for the "OpenAI Chat Model" node.

14. **Testing:**  
    - Trigger "Get Basic Information" webhook and submit form.  
    - Complete "Get Business Overview" form.  
    - Verify AI analysis generates correct unanswered questions.  
    - Complete dynamically generated "Clarification Questions" form.  
    - Confirm completion message appears.

---

### 5. General Notes & Resources

| Note Content                                                                                                                               | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| This workflow is designed to optimize user experience by avoiding redundant questions in multi-page forms using AI analysis.              | Workflow purpose and design rationale.                                                          |
| Setup requires valid OpenAI API credentials and webhook accessibility for form nodes.                                                      | Setup instructions.                                                                             |
| Prompt in "Analyse Response" node can be customized to fit different business contexts or question sets.                                  | Customization tip for adapting workflow.                                                       |
| Next steps include adding email notifications to form owners and further AI analysis to qualify leads and offer appointment booking.     | Suggested workflow enhancements.                                                               |
| For detailed explanation and use case, refer to the sticky note titled "Avoid Asking Redundant Questions with Dynamically Generated Forms". | Sticky Note4 content within the workflow.                                                      |

---

This documentation provides a complete, structured reference for understanding, reproducing, and modifying the "Dynamic Form with AI" workflow. It covers all nodes, their roles, configurations, and interconnections, enabling advanced users and AI agents to work effectively with the workflow.