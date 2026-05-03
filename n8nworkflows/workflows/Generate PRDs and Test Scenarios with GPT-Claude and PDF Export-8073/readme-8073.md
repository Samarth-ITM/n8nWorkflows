Generate PRDs and Test Scenarios with GPT/Claude and PDF Export

https://n8nworkflows.xyz/workflows/generate-prds-and-test-scenarios-with-gpt-claude-and-pdf-export-8073


# Generate PRDs and Test Scenarios with GPT/Claude and PDF Export

### 1. Workflow Overview

This workflow automates the generation of a **Product Requirements Document (PRD)** and corresponding **test scenarios** from structured user input, then produces a formatted PDF document for download. It is designed to streamline documentation tasks for product managers, business analysts, QA teams, and startup founders.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Collect structured product details via a web form.
- **1.2 PRD Generation:** Use OpenRouter GPT-4 model to generate a professional PRD in Markdown format from the input.
- **1.3 Test Scenario Generation:** Use OpenRouter Claude model to create Gherkin-style test scenarios based on the PRD.
- **1.4 Data Merging:** Combine the PRD and test scenarios into a single Markdown document.
- **1.5 PDF Creation:** Send the merged document to APITemplate.io to generate a polished PDF.
- **1.6 Output Delivery:** Provide the generated PDF file for download via a form completion node.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
  Captures product details through a web form, triggering the workflow with structured data including product name, overview, target audience, goals, and functional requirements.

- **Nodes Involved:**  
  - Get User Input (Form Trigger)

- **Node Details:**

  - **Node Name:** Get User Input  
    - **Type:** Form Trigger  
    - **Technical Role:** Entry point receiving product info from user input via a web form webhook.  
    - **Configuration:**  
      - Form titled "Technical Document Requirement"  
      - Fields: Product Name, Product Overview (required), Target Audience, Goals & Objectives, Functional Requirements  
      - Response mode set to return data from the last node (for downstream nodes)  
      - Webhook ID configured for external triggering  
    - **Input/Output:** No input; outputs JSON object with user-submitted data.  
    - **Potential Failures:** Form submission errors, missing required fields, webhook connectivity issues.

#### 1.2 PRD Generation

- **Overview:**  
  Generates a structured PRD in Markdown format based on user input using the OpenRouter GPT-4 LLM model.

- **Nodes Involved:**  
  - OpenRouter Chat Model (GPT-4)  
  - PRD LLM Chain

- **Node Details:**

  - **Node Name:** OpenRouter Chat Model  
    - **Type:** LangChain OpenRouter Chat Model (LLM)  
    - **Technical Role:** Provides GPT-4 model for natural language generation  
    - **Configuration:**  
      - Model: openai/gpt-4  
      - Credentials: OpenRouter API key  
      - No additional options specified  
    - **Input/Output:** Receives prompt text from PRD LLM Chain, outputs generated text for PRD.  
    - **Edge Cases:** API rate limits, authentication failures, model service downtime.

  - **Node Name:** PRD LLM Chain  
    - **Type:** LangChain Chain LLM  
    - **Technical Role:** Constructs prompt with user input and sends to GPT-4 model to generate PRD.  
    - **Configuration:**  
      - Prompt includes product fields (Product Name, Overview, Audience, Goals, Functional Requirements) with fallback defaults if fields are missing.  
      - Uses a detailed prompt instructing the model to produce a professional PRD in Markdown format with sections like Overview, Goals, Functional Requirements, etc.  
    - **Expressions:** Uses expressions to extract JSON fields and format the date from submission timestamp.  
    - **Input:** Receives user input JSON from Form Trigger.  
    - **Output:** Outputs Markdown PRD text.  
    - **Failure Modes:** Missing or malformed input data, expression evaluation errors, LLM response errors.

#### 1.3 Test Scenario Generation

- **Overview:**  
  Takes the generated PRD Markdown text and uses the Claude LLM to create corresponding test scenarios and Gherkin-style test cases.

- **Nodes Involved:**  
  - OpenRouter Chat Model1 (Claude)  
  - Test Case LLM Chain

- **Node Details:**

  - **Node Name:** OpenRouter Chat Model1  
    - **Type:** LangChain OpenRouter Chat Model (LLM)  
    - **Technical Role:** Provides Anthropic Claude model for natural language generation of test cases  
    - **Configuration:**  
      - Model: anthropic/claude-3.7-sonnet  
      - Credentials: OpenRouter API key (same as GPT node)  
    - **Input/Output:** Receives prompt text from Test Case LLM Chain, outputs generated test scenarios.  
    - **Edge Cases:** Same as GPT LLM node (API errors, rate limits).

  - **Node Name:** Test Case LLM Chain  
    - **Type:** LangChain Chain LLM  
    - **Technical Role:** Constructs prompt with PRD Markdown text to generate test scenarios in Gherkin language.  
    - **Configuration:**  
      - Uses PRD Markdown text as input  
      - Prompt instructs generation of test scenarios and test cases in Gherkin syntax  
    - **Input:** Receives PRD text from PRD LLM Chain output  
    - **Output:** Outputs test scenario text in Gherkin format  
    - **Failure Modes:** Empty or malformed PRD text, LLM response errors.

#### 1.4 Data Merging

- **Overview:**  
  Combines the PRD text and the generated test scenarios into a single Markdown document for PDF generation.

- **Nodes Involved:**  
  - Merge PRD and Test Case (Set Node)

- **Node Details:**

  - **Node Name:** Merge PRD and Test Case  
    - **Type:** Set Node  
    - **Technical Role:** Concatenates the PRD Markdown and test scenarios into a single text field.  
    - **Configuration:**  
      - Assigns new field `text` concatenating PRD text and test case text with a newline separator  
      - Uses expressions to access previous nodes’ outputs (`$('PRD LLM Chain').item.json.text` and current JSON `text`)  
    - **Input:** Receives PRD and test case text JSON objects  
    - **Output:** Single JSON object with merged text  
    - **Failure Modes:** Missing or undefined text fields, expression evaluation failures.

#### 1.5 PDF Creation

- **Overview:**  
  Sends the merged Markdown document to APITemplate.io to generate a PDF file using a predefined template.

- **Nodes Involved:**  
  - Create Document in PDF (APITemplate.io node)

- **Node Details:**

  - **Node Name:** Create Document in PDF  
    - **Type:** APITemplate.io Node  
    - **Technical Role:** Converts Markdown text to a formatted PDF document using an APITemplate.io template.  
    - **Configuration:**  
      - Template ID: e1277b23d41c334e (predefined PDF template)  
      - Filename: PRD_abc.pdf (default output filename)  
      - Input property: `markdown` set to merged text field from previous node  
      - Download enabled to fetch generated PDF binary data  
      - Credentials: APITemplate.io API key  
    - **Input:** Receives merged Markdown text in JSON  
    - **Output:** Binary PDF file data  
    - **Failure Modes:** API errors, template ID invalid, network issues, auth failures.

#### 1.6 Output Delivery

- **Overview:**  
  Provides the generated PDF file back to the user via a form completion node allowing download.

- **Nodes Involved:**  
  - Let User Download (Form Completion Node)

- **Node Details:**

  - **Node Name:** Let User Download  
    - **Type:** Form Node (Completion)  
    - **Technical Role:** Sends the generated PDF binary back to the user interface for download after workflow completion.  
    - **Configuration:**  
      - Operation: completion  
      - Responds with binary data (`returnBinary`)  
      - Completion title/message set to inform user PDF is ready  
      - Input data field name set to `=data` (mapped binary from PDF node)  
      - Webhook ID configured for communication with original form  
    - **Input:** Receives binary PDF file from Create Document in PDF node  
    - **Output:** HTTP response delivering binary file to user  
    - **Failure Modes:** Binary data missing, webhook timeout, user disconnect.

---

### 3. Summary Table

| Node Name              | Node Type                          | Functional Role                                  | Input Node(s)             | Output Node(s)          | Sticky Note                                                                                   |
|------------------------|----------------------------------|-------------------------------------------------|---------------------------|-------------------------|------------------------------------------------------------------------------------------------|
| Get User Input          | Form Trigger                     | Collect user product details                     |                           | PRD LLM Chain           |                                                                                                |
| OpenRouter Chat Model   | LangChain LLM (OpenRouter GPT)  | Generate PRD Markdown from user input           | PRD LLM Chain             | PRD LLM Chain           |                                                                                                |
| PRD LLM Chain           | LangChain Chain LLM              | Format prompt and request PRD text generation    | Get User Input            | Test Case LLM Chain     |                                                                                                |
| OpenRouter Chat Model1  | LangChain LLM (OpenRouter Claude)| Generate test scenarios in Gherkin from PRD     | Test Case LLM Chain       | Test Case LLM Chain     |                                                                                                |
| Test Case LLM Chain     | LangChain Chain LLM              | Format prompt and request test case generation   | PRD LLM Chain             | Merge PRD and Test Case |                                                                                                |
| Merge PRD and Test Case | Set Node                        | Concatenate PRD and test scenarios text          | Test Case LLM Chain       | Create Document in PDF  |                                                                                                |
| Create Document in PDF  | APITemplate.io Node              | Convert Markdown to PDF document                  | Merge PRD and Test Case   | Let User Download       |                                                                                                |
| Let User Download       | Form Completion Node             | Deliver PDF file to user for download            | Create Document in PDF    |                         |                                                                                                |
| Sticky Note             | Sticky Note                     | Documentation - workflow description and usage   |                           |                         | ### 📒 Generate **Product Requirements Document (PRD)** and **test scenarios** form input to PDF with OpenRouter and APITemplate.io (See full note content) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the Form Trigger node**  
   - Type: Form Trigger  
   - Configure webhook for external submissions (copy webhook URL for use)  
   - Form title: "Technical Document Requirement"  
   - Add fields:  
     - Product Name (text)  
     - Product Overview (textarea, required)  
     - Target Audience (text)  
     - Goals & Objectives (text)  
     - Functional Requirements (text)  
   - Set response mode to “last node”  
   - Save node as "Get User Input"

2. **Create OpenRouter Chat Model node (GPT-4)**  
   - Type: LangChain OpenRouter Chat Model  
   - Set model to "openai/gpt-4"  
   - Add OpenRouter API credentials  
   - Save as "OpenRouter Chat Model"

3. **Create PRD LLM Chain node**  
   - Type: LangChain Chain LLM  
   - Text input:  
     ```
     =Product Name - {{ $json['Product Name'] || 'Not Available'}}
     Product Overview  - {{ $json['Product Overview'] || 'Not Available' }}
     Target Audience - {{ $json['Target Audience'] || 'Not Available' }}
     Goal & Objective - {{ $json['Goals & Objectives'] || 'Not Available' }}
     Functional Requirements - {{ $json['Functional Requirements'] || 'Not Available' }}
     Date: {{ $json.submittedAt.split("T")[0] }}
     Created By - {{ $json['Created By'] || 'Not Available' }}
     ```
   - Add system prompt instructing expert product manager and technical writer to generate a Markdown PRD with detailed sections, bullet points, and formatting rules (see node prompt above for full text)  
   - Connect "Get User Input" node output to this node’s input  
   - Under AI Language Model option, select "OpenRouter Chat Model"  
   - Save as "PRD LLM Chain"

4. **Create OpenRouter Chat Model1 node (Claude)**  
   - Type: LangChain OpenRouter Chat Model  
   - Set model to "anthropic/claude-3.7-sonnet"  
   - Add same OpenRouter API credentials  
   - Save as "OpenRouter Chat Model1"

5. **Create Test Case LLM Chain node**  
   - Type: LangChain Chain LLM  
   - Input text: `={{ $json.text }}` (output from PRD LLM Chain)  
   - Prompt: "Using the user input PRD document create Test scenario and test case in gherkin language."  
   - Connect "PRD LLM Chain" output to this node’s input  
   - Under AI Language Model option, select "OpenRouter Chat Model1"  
   - Save as "Test Case LLM Chain"

6. **Create Merge PRD and Test Case node**  
   - Type: Set Node  
   - Create new field `text` with value:  
     ```
     ={{ $('PRD LLM Chain').item.json.text }}
     {{ $json.text }}
     ```  
   - Connect "Test Case LLM Chain" output here  
   - Save as "Merge PRD and Test Case"

7. **Create Create Document in PDF node**  
   - Type: APITemplate.io node  
   - Resource: PDF  
   - PDF Template ID: set to your APITemplate.io PDF template ID (e.g., e1277b23d41c334e)  
   - Filename: "PRD_abc.pdf"  
   - Property `markdown` set to: `={{ $json.text }}`  
   - Enable download to get binary PDF output  
   - Add APITemplate.io credentials  
   - Connect "Merge PRD and Test Case" output to this node  
   - Save as "Create Document in PDF"

8. **Create Let User Download node**  
   - Type: Form Completion Node  
   - Operation: completion  
   - Respond with: returnBinary  
   - Completion Title: "PRD Document is ready"  
   - Completion Message: "Process completed file will be downloaded!"  
   - Input Data Field Name: `=data` (binary PDF from previous node)  
   - Configure webhook for form response  
   - Connect "Create Document in PDF" output to this node  
   - Save as "Let User Download"

9. **Connect the nodes in order:**  
   - Get User Input → PRD LLM Chain → Test Case LLM Chain → Merge PRD and Test Case → Create Document in PDF → Let User Download

10. **Credentials:**  
    - Setup OpenRouter API credentials with your API key for both LLM nodes.  
    - Setup APITemplate.io API credentials with your account API key.

11. **Test the workflow:**  
    - Submit the form with valid product data.  
    - Confirm PRD and test scenarios are generated.  
    - Verify PDF is created and downloadable.

---

### 5. General Notes & Resources

| Note Content                                                                                                                       | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| 📒 This workflow generates **Product Requirements Documents (PRDs)** and **test scenarios** from structured form inputs using OpenRouter GPT/Claude models and APITemplate.io for PDF export. It is targeted at product managers, QA teams, and startup founders for rapid documentation creation. | See Sticky Note node content in workflow for detailed overview and instructions.                 |
| ⚡ Requirements: OpenRouter API Key (or any LLM provider) and APITemplate.io account are mandatory for this workflow to function. |                                                                                                |
| 🎯 Use Cases: Rapid PRD drafting, automated test scenario generation, and standardized documentation workflows.                  |                                                                                                |
| Customization suggestions: Edit prompt texts in LLM Chain nodes, swap PDF templates in APITemplate.io, or extend with Slack/Notion integrations. |                                                                                                |

---

**Disclaimer:**  
The text provided is derived exclusively from an n8n automated workflow. It strictly complies with content policies and contains no illegal, offensive, or protected material. All data processed is legal and publicly available.