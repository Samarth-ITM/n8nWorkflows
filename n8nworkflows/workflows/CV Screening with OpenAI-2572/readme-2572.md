CV Screening with OpenAI

https://n8nworkflows.xyz/workflows/cv-screening-with-openai-2572


# CV Screening with OpenAI

### 1. Workflow Overview

This workflow automates the screening of resumes (CVs) using OpenAI’s language model, designed specifically for recruitment professionals, HR teams, and hiring managers. It processes candidate CV files by downloading them from a direct URL, extracting textual content, and then analyzing this text in the context of a provided job description through OpenAI’s API. The output includes a suitability matching percentage, a concise summary, and detailed reasons for and against the candidate’s fit for the role.

**Logical Blocks:**

- **1.1 Input Initialization**: Manual trigger and variable setup including CV URL and job description.
- **1.2 Resume Retrieval and Extraction**: Downloading the candidate’s CV file and extracting its text content.
- **1.3 AI Analysis**: Sending extracted text and job description to OpenAI for structured evaluation.
- **1.4 Output Processing**: Parsing and formatting OpenAI’s JSON schema response for downstream use.
- **1.5 Documentation and Guidance**: Sticky notes for setup instructions, video guide references, and workflow context.

---

### 2. Block-by-Block Analysis

#### 2.1 Input Initialization

- **Overview:**  
  This block initializes the workflow manually and sets key variables such as the CV file URL, job description, the prompt for OpenAI, and the expected JSON schema for response parsing.

- **Nodes Involved:**  
  - When clicking ‘Test workflow’ (Manual Trigger)  
  - Set Variables

- **Node Details:**  

  - **When clicking ‘Test workflow’**  
    - Type: Manual Trigger  
    - Role: Entry point for manual execution during testing or demonstration.  
    - Configuration: No parameters; triggers the flow on manual interaction.  
    - Inputs: None  
    - Outputs: Connected to "Set Variables"  
    - Edge Cases: None significant; if not triggered, workflow does not start.

  - **Set Variables**  
    - Type: Set node (assigns static data)  
    - Role: Define and output key workflow parameters:  
      - `file_url`: Direct URL to candidate’s CV PDF file.  
      - `job_description`: Text describing the job to match against.  
      - `prompt`: Instructions for OpenAI on how to analyze the CV.  
      - `json_schema`: JSON Schema defining the expected structured response.  
    - Configuration: Hardcoded example values for testing; replace with dynamic inputs in production.  
    - Inputs: Manual Trigger  
    - Outputs: Connected to "Download File"  
    - Edge Cases: Incorrect or invalid URLs or job descriptions can cause failures downstream; prompt clarity affects AI response quality.

---

#### 2.2 Resume Retrieval and Extraction

- **Overview:**  
  Downloads the CV file from the provided URL and extracts the textual content from the PDF for further analysis.

- **Nodes Involved:**  
  - Download File  
  - Extract Document PDF

- **Node Details:**  

  - **Download File**  
    - Type: HTTP Request  
    - Role: Fetch the candidate’s resume file using the direct link provided in `file_url`.  
    - Configuration:  
      - URL: Dynamically set from `{{$json.file_url}}` from previous node.  
      - Method: GET (default)  
      - No additional options or authentication specified (assumes public accessible URL).  
    - Inputs: From "Set Variables"  
    - Outputs: Passes binary file data to "Extract Document PDF"  
    - Edge Cases:  
      - URL not accessible or invalid → HTTP errors or empty response.  
      - Network timeout or slow response → timeout errors.  
      - File format not supported or corrupted files could cause extraction failure.

  - **Extract Document PDF**  
    - Type: Extract from File  
    - Role: Extracts text content from the downloaded PDF file.  
    - Configuration: Operation set to “pdf” to handle PDF file extraction.  
    - Inputs: Binary file data from "Download File" node.  
    - Outputs: Extracted text content (`text` field) forwarded to "OpenAI - Analyze CV".  
    - Edge Cases:  
      - PDF with complex formatting, images, or scanned pages may yield incomplete or poor text extraction.  
      - Unsupported file types or corrupted PDFs cause extraction failure.

---

#### 2.3 AI Analysis

- **Overview:**  
  Sends the extracted text and job description along with a prompt to OpenAI’s chat completion API, requesting a structured candidate evaluation response conforming to a JSON schema.

- **Nodes Involved:**  
  - OpenAI - Analyze CV  
  - Set Variables (provides prompt and schema references)

- **Node Details:**  

  - **OpenAI - Analyze CV**  
    - Type: HTTP Request (with OpenAI credentials)  
    - Role: Calls OpenAI’s GPT-4o-mini model to evaluate the CV against the job description.  
    - Configuration:  
      - URL: `https://api.openai.com/v1/chat/completions`  
      - Method: POST  
      - Body: JSON containing:  
        - Model: `gpt-4o-mini`  
        - Messages:  
          - System message: prompt string from "Set Variables" node.  
          - User message: the extracted CV text, URL-encoded and stringified.  
        - `response_format`:  
          - type: `json_schema`  
          - json_schema: JSON schema from "Set Variables" to enforce structured response.  
      - Authentication: Uses predefined OpenAI API Credential stored in n8n.  
    - Inputs: Extracted text from "Extract Document PDF" node, plus prompt and schema from "Set Variables" (via expressions).  
    - Outputs: Raw JSON response from OpenAI.  
    - Edge Cases:  
      - API key invalid or quota exceeded → authentication errors.  
      - Network issues or API downtime → request timeouts or failures.  
      - Model returns malformed or incomplete JSON → parsing errors downstream.  
      - Prompt ambiguity → inaccurate or irrelevant responses.

---

#### 2.4 Output Processing

- **Overview:**  
  Parses the JSON response string from OpenAI into a structured JSON object for easier consumption and further processing.

- **Nodes Involved:**  
  - Parsed JSON

- **Node Details:**  

  - **Parsed JSON**  
    - Type: Set node  
    - Role: Converts OpenAI’s raw text content (stringified JSON) into a JSON object.  
    - Configuration:  
      - Uses expression: `={{ JSON.parse($json.choices[0].message.content) }}`  
      - Assigns parsed object to `json_parsed` property.  
    - Inputs: Output from "OpenAI - Analyze CV" node.  
    - Outputs: Parsed structured data ready for storage or downstream use.  
    - Edge Cases:  
      - If OpenAI response is not valid JSON, parsing will fail causing errors.  
      - If response format changes upstream, this node may need adjustment.

---

#### 2.5 Documentation and Guidance

- **Overview:**  
  A series of sticky notes providing setup instructions, workflow purpose, usage tips, and links to video guides.

- **Nodes Involved:**  
  - Sticky Note  
  - Sticky Note1  
  - Sticky Note2  
  - Sticky Note5  
  - Sticky Note6

- **Node Details:**  

  - Sticky Note  
    - Provides instruction to "Add direct link to CV and Job description" near the variable setup.  
  - Sticky Note1  
    - Notes to "Replace OpenAI connection" — reminder to update API credentials as needed.  
  - Sticky Note2  
    - Links a YouTube video setup guide (8 minutes) with thumbnail image.  
  - Sticky Note5  
    - Summarizes the workflow steps in setup form: download file, extract data, send to OpenAI, save results.  
  - Sticky Note6  
    - Branding and credits, workflow overview, and detailed explanation of functionality and target users. Includes logos and links to author and community.  

---

### 3. Summary Table

| Node Name              | Node Type          | Functional Role                         | Input Node(s)                  | Output Node(s)           | Sticky Note                                        |
|------------------------|--------------------|---------------------------------------|-------------------------------|--------------------------|---------------------------------------------------|
| When clicking ‘Test workflow’ | Manual Trigger     | Entry point for manual workflow start | None                          | Set Variables             |                                                   |
| Set Variables          | Set                | Define CV URL, job description, prompt, JSON schema | When clicking ‘Test workflow’ | Download File             | **Add direct link to CV and Job description**     |
| Download File          | HTTP Request       | Downloads candidate CV file from URL  | Set Variables                 | Extract Document PDF      |                                                   |
| Extract Document PDF   | Extract from File  | Extracts text content from downloaded PDF | Download File                | OpenAI - Analyze CV       |                                                   |
| OpenAI - Analyze CV    | HTTP Request       | Sends CV text and job description to OpenAI for evaluation | Extract Document PDF          | Parsed JSON               | **Replace OpenAI connection**                      |
| Parsed JSON            | Set                | Parses OpenAI JSON string response to object | OpenAI - Analyze CV            | None                     |                                                   |
| Sticky Note            | Sticky Note        | Instruction to add CV and job description links | None                          | None                     | **Add direct link to CV and Job description**     |
| Sticky Note1           | Sticky Note        | Reminder to update OpenAI credentials | None                          | None                     | **Replace OpenAI connection**                      |
| Sticky Note2           | Sticky Note        | Link to 8-minute video setup guide    | None                          | None                     | [YouTube Setup Video](https://youtu.be/TWuI3dOcn0E) |
| Sticky Note5           | Sticky Note        | Setup summary steps                   | None                          | None                     | Summary of workflow setup steps                    |
| Sticky Note6           | Sticky Note        | Workflow overview, credits, branding | None                          | None                     | Workflow purpose, author, and community credits   |

---

### 4. Reproducing the Workflow from Scratch

1. **Create a new workflow in n8n.**

2. **Add a Manual Trigger node:**
   - Name: `When clicking ‘Test workflow’`
   - Default settings; this is the workflow’s starting point.

3. **Add a Set node:**
   - Name: `Set Variables`
   - Connect output of Manual Trigger to this node.
   - Add the following fields with static values (replace with dynamic inputs as needed):  
     - `file_url`: string; example URL to a PDF CV (e.g., `https://.../software_engineer_resume_example.pdf`)  
     - `job_description`: string; detailed job description text.  
     - `prompt`: string; detailed instructions for OpenAI regarding CV evaluation.  
     - `json_schema`: string; JSON schema defining the structure of OpenAI’s expected response.  
   - Ensure all fields are correctly spelled and formatted.

4. **Add an HTTP Request node:**
   - Name: `Download File`
   - Connect output of Set Variables node to this node.
   - Configure:  
     - Method: GET (default)  
     - URL: Use expression `={{ $json.file_url }}` to reference the CV URL from Set Variables.  
     - No authentication required if the URL is publicly accessible.

5. **Add Extract from File node:**
   - Name: `Extract Document PDF`
   - Connect output of Download File node to this node.
   - Configure:  
     - Operation: `pdf` to extract text from PDF file.  
     - Input: Binary file data from Download File node.

6. **Add HTTP Request node for OpenAI API:**
   - Name: `OpenAI - Analyze CV`
   - Connect output of Extract Document PDF node to this node.
   - Configure:  
     - Method: POST  
     - URL: `https://api.openai.com/v1/chat/completions`  
     - Authentication: Use predefined OpenAI credentials (set up separately in n8n credentials).  
     - Body Content Type: JSON  
     - Body Parameters:  
       - `model`: `"gpt-4o-mini"`  
       - `messages`: Array with two messages:  
         - System message with prompt from Set Variables node: `{{ $('Set Variables').item.json.prompt }}`  
         - User message with encoded extracted CV text: `{{ JSON.stringify(encodeURIComponent($json.text)) }}`  
       - `response_format`: Object specifying JSON schema type and the schema itself, from Set Variables node:  
         `{ "type": "json_schema", "json_schema": {{ $('Set Variables').item.json.json_schema }} }`  
     - Ensure `sendBody` is true and `specifyBody` is set to JSON.

7. **Add a Set node to parse OpenAI’s response:**
   - Name: `Parsed JSON`
   - Connect output of OpenAI - Analyze CV node to this node.
   - Configure:  
     - Add an assignment that parses the first choice’s message content:  
       `json_parsed = JSON.parse($json.choices[0].message.content)`

8. **Add sticky notes for documentation (optional but recommended):**
   - Add sticky notes near relevant nodes with instructions such as:  
     - “Add direct link to CV and Job description” near Set Variables.  
     - “Replace OpenAI connection” near OpenAI node reminder.  
     - Link to the 8-minute YouTube video guide.  
     - Workflow overview and credits.

9. **Set up OpenAI credentials in n8n:**
   - Go to Credentials in n8n.  
   - Add new credentials for OpenAI API with a valid API key.

10. **Test the workflow:**
    - Trigger the manual node.  
    - Verify the CV is downloaded, text extracted, and OpenAI returns a parsed JSON evaluation.

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                              | Context or Link                                                                                   |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Detailed video guide showing the entire build process of the resume analyzer.                                                                                                                                                                             | [YouTube Video (8 min)](https://youtu.be/TWuI3dOcn0E)                                           |
| Workflow authored by Mark Shcherbakov from the community 5minAI.                                                                                                                                                                                          | [Mark Shcherbakov LinkedIn](https://www.linkedin.com/in/marklowcoding/), [5minAI Community](https://www.skool.com/5minai-2861) |
| Workflow ideal for automating initial screening of CVs for recruitment agencies and HR professionals to save time and reduce bias.                                                                                                                        | Workflow description and purpose notes                                                           |
| Reminder to replace the OpenAI API key in the workflow with your own for proper authentication and quota management.                                                                                                                                       | Sticky Note near OpenAI node                                                                     |
| For best results, CV files should be clean PDFs with selectable text; scanned images or complex formatting may reduce extraction quality.                                                                                                                 | Extraction node operation notes                                                                   |
| JSON Schema used ensures consistent and structured output from OpenAI, facilitating integration with databases or downstream systems.                                                                                                                     | Set Variables node configuration                                                                 |

---

This documentation provides a comprehensive understanding of the "CV Screening with OpenAI" workflow, enabling users and agents to comprehend, reproduce, and extend the automation efficiently.