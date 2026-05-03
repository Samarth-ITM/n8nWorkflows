Resume Reviewer with GPT-4o-mini and Human Validation via gotoHuman

https://n8nworkflows.xyz/workflows/resume-reviewer-with-gpt-4o-mini-and-human-validation-via-gotohuman-7605


# Resume Reviewer with GPT-4o-mini and Human Validation via gotoHuman

---

### 1. Workflow Overview

This workflow implements a **Resume Reviewer** system that combines AI-powered scoring with human validation. It is designed for scenarios where PDF resumes are submitted and need to be assessed against a predefined job description. The system automatically extracts text from uploaded resumes, evaluates their relevance to the job description using GPT-4o-mini, generates a match score and summary, and then routes the result to a human reviewer via **gotoHuman** for approval or further feedback.

The workflow is logically divided into the following blocks:

- **1.1 Input Reception:** Receives PDF resume uploads via an n8n form trigger.
- **1.2 Resume Text Extraction:** Extracts text content from the uploaded PDF file.
- **1.3 Job Description Setup:** Sets a static job description text used for comparison.
- **1.4 AI Processing:** Uses GPT-4o-mini to compare the resume text to the job description, producing a summary and a match score.
- **1.5 Structured Output Parsing:** Ensures the AI output matches the expected JSON schema.
- **1.6 Human Review via gotoHuman:** Sends the AI-generated summary, score, and resume text to a human reviewer and waits for their validation.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:** This block captures the resume submission from users. It uses an n8n form trigger node configured to accept only PDF files.
- **Nodes Involved:** `Resume Submission`
- **Node Details:**

  - **Resume Submission**
    - Type: Form Trigger node
    - Role: Entry point to capture resume PDF uploads from users.
    - Configuration:
      - Form title: "Resume - PDF Only"
      - Form fields: single file upload field accepting `.pdf` files only.
    - Inputs: External HTTP webhook (form submission)
    - Outputs: Binary data containing the uploaded PDF file.
    - Edge cases:
      - Upload of non-PDF files will be rejected by form configuration.
      - Large file uploads might cause timeouts.
    - Notes: Webhook ID uniquely identifies this form trigger.

#### 1.2 Resume Text Extraction

- **Overview:** Converts the uploaded PDF resume into plain text for AI processing.
- **Nodes Involved:** `Extract from File`
- **Node Details:**

  - **Extract from File**
    - Type: Extract from File node
    - Role: Extracts text from the binary PDF data.
    - Configuration:
      - Operation: PDF text extraction
      - Binary Property Name: "Resume" (matches the uploaded file's binary property)
    - Inputs: Binary PDF file from `Resume Submission`
    - Outputs: JSON containing extracted text in `item.json.text`
    - Edge cases:
      - Poorly formatted or scanned PDFs may yield incomplete text.
      - Extraction failure if binary data is corrupted or empty.

#### 1.3 Job Description Setup

- **Overview:** Provides a static job description text used as a reference for resume evaluation.
- **Nodes Involved:** `Set Job Description`
- **Node Details:**

  - **Set Job Description**
    - Type: Set node
    - Role: Assigns a detailed job description string to a JSON property `Job Description`.
    - Configuration:
      - Static multiline string describing an "n8n Specialist" job role.
    - Inputs: Output from `Extract from File`
    - Outputs: JSON with `Job Description` field added
    - Edge cases:
      - None expected; static content.

#### 1.4 AI Processing

- **Overview:** Uses GPT-4o-mini to compare the extracted resume text to the job description, generating a summary and a numeric match score (0-100).
- **Nodes Involved:** `OpenAI Chat Model9`, `Review Resume`, `Structured Output Parser7`
- **Node Details:**

  - **OpenAI Chat Model9**
    - Type: LangChain OpenAI Chat Model node
    - Role: Sends prompt to GPT-4o-mini model.
    - Configuration:
      - Model: `gpt-4o-mini`
      - No additional options configured.
      - Uses OpenAI API credential configured in n8n.
    - Inputs: None directly; connected as an AI model invoked by `Review Resume`.
    - Outputs: AI model response.
    - Edge cases:
      - API authentication failure.
      - Rate limiting, timeouts.
      - Model response errors or malformed output.

  - **Review Resume**
    - Type: LangChain Agent node
    - Role: Orchestrates the prompt to compare resume and job description, requesting score and summary.
    - Configuration:
      - Text prompt includes variables: `Job Description` and extracted resume text.
      - System message instructs model to output JSON with `summary` and `score`.
      - Output parser enabled.
    - Inputs: Output from `Set Job Description` (job description), and connected to AI model and output parser.
    - Outputs: Structured JSON with `summary` and `score`.
    - Edge cases:
      - Expression failures if variables are missing or undefined.
      - Unexpected AI output format.

  - **Structured Output Parser7**
    - Type: LangChain Structured Output Parser node
    - Role: Validates that AI output conforms to expected JSON schema.
    - Configuration:
      - JSON schema example with `summary` (string) and `score` (number).
    - Inputs: AI output from `OpenAI Chat Model9`
    - Outputs: Parsed JSON for downstream use.
    - Edge cases:
      - Parsing failure if AI output is malformed or deviates from schema.

#### 1.5 Human Review via gotoHuman

- **Overview:** Sends the AI-generated summary, score, and full resume text to a human review platform (gotoHuman), then waits for human feedback before continuing.
- **Nodes Involved:** `Send to GoToHuman`
- **Node Details:**

  - **Send to GoToHuman**
    - Type: gotoHuman node (community node)
    - Role: Submits review request to gotoHuman platform with mapped fields; waits for human completion.
    - Configuration:
      - Credentials: gotoHuman API key stored in n8n credentials.
      - Review Template ID: A configured template ID (e.g., `SLFm3wk8I1kGuEmRbmIr`).
      - Fields mapping:
        - `Resume`: mapped from extracted text (`Extract from File`)
        - `Summary`: mapped from AI output summary (`$json.output.summary`)
        - `Rating`: mapped from AI output score (`$json.output.score`)
      - Schema defines expected fields: Resume (string), Summary (string), Rating (number).
      - Waits for human to complete the review before resuming.
    - Inputs: Output from `Review Resume`
    - Outputs: None connected (terminal node).
    - Edge cases:
      - API authentication failure.
      - Network issues.
      - Human reviewer delays or failure to respond.
      - Inconsistent field mapping or schema mismatch.
    - Notes: The node acts as a manual validation point, integrating human-in-the-loop.

#### Additional Notes on Sticky Notes

- Several `Sticky Note` nodes provide instructions and context:

  - `Sticky Note29` details setup instructions for OpenAI and gotoHuman connections, including billing and template setup.
  - `Sticky Note30` describes the overall workflow purpose and flow.
  - `Sticky Note31` and `Sticky Note32` reinforce configuration steps for OpenAI and gotoHuman respectively.

---

### 3. Summary Table

| Node Name            | Node Type                              | Functional Role                        | Input Node(s)           | Output Node(s)         | Sticky Note                                                                                          |
|----------------------|---------------------------------------|-------------------------------------|------------------------|------------------------|----------------------------------------------------------------------------------------------------|
| Resume Submission    | Form Trigger                          | Receives PDF resume uploads          | -                      | Extract from File       |                                                                                                    |
| Extract from File    | Extract from File                     | Extracts text from PDF file          | Resume Submission       | Set Job Description     |                                                                                                    |
| Set Job Description  | Set                                  | Sets static job description string  | Extract from File       | Review Resume           |                                                                                                    |
| OpenAI Chat Model9   | LangChain OpenAI Chat Model           | Calls GPT-4o-mini AI model           | -                      | Review Resume           | ### 1️⃣ Set Up OpenAI Connection (see Sticky Note31 for details)                                  |
| Review Resume        | LangChain Agent                      | Compares resume to job description, outputs summary and score | Set Job Description, OpenAI Chat Model9, Structured Output Parser7 | Send to GoToHuman           |                                                                                                    |
| Structured Output Parser7 | LangChain Structured Output Parser | Validates AI output JSON schema     | OpenAI Chat Model9      | Review Resume           |                                                                                                    |
| Send to GoToHuman    | gotoHuman                            | Sends review request to human, waits for approval | Review Resume           | -                      | ### 2️⃣ gotoHuman — Install, Configure, and Map (see Sticky Note32 and Sticky Note29 for details) |
| Sticky Note29        | Sticky Note                         | Setup instructions for OpenAI and gotoHuman | -                      | -                      | Contains detailed setup steps and contact info                                                    |
| Sticky Note30        | Sticky Note                         | Workflow description overview        | -                      | -                      | Summarizes workflow purpose                                                                       |
| Sticky Note31        | Sticky Note                         | OpenAI setup instructions            | -                      | -                      |                                                                                                    |
| Sticky Note32        | Sticky Note                         | gotoHuman setup instructions         | -                      | -                      |                                                                                                    |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a Form Trigger node named `Resume Submission`:**
   - Set form title to `Resume - PDF Only`.
   - Add a single file upload field:
     - Field Label: `Resume`
     - Accept File Types: `.pdf`
   - This node will provide a webhook URL to receive form submissions.

2. **Add an `Extract from File` node named `Extract from File`:**
   - Configure operation as `pdf`.
   - Set Binary Property Name to `Resume` (matching the form field).
   - Connect the output of `Resume Submission` to this node.

3. **Add a `Set` node named `Set Job Description`:**
   - Add a string field named `Job Description`.
   - Paste the full static job description text for the "n8n Specialist" role.
   - Connect the output of `Extract from File` to this node.

4. **Add a LangChain OpenAI Chat Model node named `OpenAI Chat Model9`:**
   - Select model `gpt-4o-mini`.
   - Add OpenAI API credentials (configured with your OpenAI API key).
   - No extra options needed.
   - This node will be used as the AI language model for the agent.

5. **Add a LangChain Structured Output Parser node named `Structured Output Parser7`:**
   - Define JSON schema example:
     ```json
     {
       "summary": "summary",
       "score": 50
     }
     ```
   - This node will parse and validate the AI output.

6. **Add a LangChain Agent node named `Review Resume`:**
   - Set the prompt text to:
     ```
     Job Description: {{ $json['Job Description'] }}
     Resume: {{ $('Extract from File').item.json.text }}
     ```
   - Set system message to:
     ```
     compare the resume against the job description and give a value between 0 and 100 for how well it matches up with the job description. Also output a summary of the resume with the points that most closely match the description. Should be one paragraph and bullet points.

     output like this..
     {
       "summary": "summary",
       "score": 50
     }
     ```
   - Enable output parser.
   - Connect input from `Set Job Description`.
   - Connect AI language model input to `OpenAI Chat Model9`.
   - Connect AI output parser input to `Structured Output Parser7`.

7. **Add a `gotoHuman` node named `Send to GoToHuman`:**
   - Install the community node `@gotohuman/n8n-nodes-gotohuman` via n8n Settings if not already installed.
   - Create and configure a gotoHuman API credential with your API key.
   - Set Review Template ID to your gotoHuman Review Template ID (e.g., `SLFm3wk8I1kGuEmRbmIr`).
   - Configure fields mapping as:
     - `Resume` → `{{$('Extract from File').item.json.text}}`
     - `Summary` → `{{$json.output.summary}}`
     - `Rating` → `{{$json.output.score}}`
   - Define schema with:
     - `Resume` (string)
     - `Summary` (string)
     - `Rating` (number)
   - Connect output from `Review Resume` to this node.

8. **Set node connections as follows:**
   - `Resume Submission` → `Extract from File`
   - `Extract from File` → `Set Job Description`
   - `Set Job Description` → `Review Resume`
   - `OpenAI Chat Model9` → AI language model input of `Review Resume`
   - `Structured Output Parser7` → AI output parser input of `Review Resume`
   - `Review Resume` → `Send to GoToHuman`

9. **Credentials Setup:**
   - Create OpenAI credentials within n8n using your OpenAI API key.
   - Create gotoHuman API credentials using your gotoHuman API key.
   - Assign these credentials to `OpenAI Chat Model9` and `Send to GoToHuman` nodes respectively.

10. **Test the workflow:**
    - Submit a PDF resume through the form webhook URL.
    - Confirm text extraction.
    - Confirm AI scoring and summary generation.
    - Confirm human review step triggers and waits for completion.

---

### 5. General Notes & Resources

| Note Content                                                                                                                        | Context or Link                                                                                           |
|------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------|
| Setup instructions for OpenAI API keys, billing, and credential configuration in n8n.                                              | [OpenAI Platform](https://platform.openai.com/api-keys), billing overview link provided in Sticky Note29.|
| gotoHuman node installation and configuration steps, including creating templates and API keys.                                    | n8n Community Nodes installation, gotoHuman API setup instructions in Sticky Note29 and Sticky Note32.   |
| Workflow description: Human-in-the-loop resume review combining AI and manual validation for accuracy and approval.               | See Sticky Note30 for high-level workflow summary.                                                       |
| Contact info for customization or consulting: Robert Breen, robert@ynteractive.com, https://ynteractive.com                        | Provided in Sticky Note29.                                                                                |

---

*Disclaimer: The provided text is sourced exclusively from an automated workflow created with n8n, adhering strictly to current content policies. It contains no illegal, offensive, or protected material. All handled data is legal and public.*