🧠 FloWatch 👁️ Analyze and Diagnose n8n Workflow Errors via OpenAI and Email

https://n8nworkflows.xyz/workflows/---flowatch-----analyze-and-diagnose-n8n-workflow-errors-via-openai-and-email-3595


# 🧠 FloWatch 👁️ Analyze and Diagnose n8n Workflow Errors via OpenAI and Email

### 1. Workflow Overview

This workflow, titled **"🧠 FloWatch 👁️ Analyze and Diagnose n8n Workflow Errors via OpenAI and Email"**, is designed to automate the detection, diagnosis, and reporting of errors occurring in n8n workflows. It targets n8n developers, automation engineers, and DevOps teams who want to streamline troubleshooting by automatically capturing workflow failures, analyzing them with AI assistance, and sending detailed, styled HTML email reports to administrators or support teams.

The workflow is logically divided into the following blocks:

- **1.1 Error Detection & Filtering:** Captures any workflow error via the Error Trigger node, filters out manual executions if desired, and retrieves detailed execution data.
- **1.2 Error Extraction & Structuring:** Processes the failed execution data to extract comprehensive error metadata and details from all nodes involved (excluding certain nodes).
- **1.3 AI-Powered Error Diagnosis:** Sends the extracted error details to OpenAI’s GPT-4o model via a LangChain agent node to generate a diagnosis, cause, and resolution.
- **1.4 Formatting & Email Reporting:** Formats the AI-generated analysis into a professional HTML email and sends it to configured recipients using Gmail SMTP credentials.

---

### 2. Block-by-Block Analysis

#### 2.1 Error Detection & Filtering

- **Overview:**  
  This block listens for any error occurring in any n8n workflow, optionally filters out manual executions to avoid noise, and fetches the full failed execution details for further processing.

- **Nodes Involved:**  
  - Error Trigger  
  - SET EMAIL  
  - Get Failed Exec  
  - Remove Manual Exec  

- **Node Details:**

  - **Error Trigger**  
    - *Type:* n8n-nodes-base.errorTrigger  
    - *Role:* Entry point that triggers the workflow on any n8n workflow error.  
    - *Configuration:* Default, listens globally to errors.  
    - *Connections:* Outputs to SET EMAIL node.  
    - *Edge Cases:* May trigger on manual executions or irrelevant workflows if not filtered.  
    - *Version:* Standard node, no special version requirements.

  - **SET EMAIL**  
    - *Type:* n8n-nodes-base.set  
    - *Role:* Defines email addresses for TO, CC, and BCC recipients.  
    - *Configuration:* Hardcoded emails (e.g., myemail@myemail.com for TO).  
    - *Connections:* Input from Error Trigger, output to Get Failed Exec.  
    - *Edge Cases:* Emails must be valid; misconfiguration leads to failed email delivery.

  - **Get Failed Exec**  
    - *Type:* n8n-nodes-base.n8n (n8n API node)  
    - *Role:* Retrieves detailed execution data for the failed workflow execution using the execution ID from the error trigger.  
    - *Configuration:* Uses execution ID from Error Trigger JSON (`={{ $('Error Trigger').item.json.execution.id }}`), requires n8n API credentials.  
    - *Connections:* Input from SET EMAIL, output to Remove Manual Exec.  
    - *Edge Cases:* API authentication errors, execution ID missing or invalid, network timeouts.

  - **Remove Manual Exec**  
    - *Type:* n8n-nodes-base.if  
    - *Role:* Filters out executions triggered manually to avoid processing non-error or test runs.  
    - *Configuration:* Checks if the execution mode string does NOT contain "manual".  
    - *Connections:* Input from Get Failed Exec, output to Extract Error Details.  
    - *Edge Cases:* If mode field missing or malformed, may fail filtering.

---

#### 2.2 Error Extraction & Structuring

- **Overview:**  
  Extracts detailed metadata and error information from the failed execution, including node names, error messages, timestamps, and excludes nodes containing "SERP" in their names.

- **Nodes Involved:**  
  - Extract Error Details  

- **Node Details:**

  - **Extract Error Details**  
    - *Type:* n8n-nodes-base.code (JavaScript code node)  
    - *Role:* Parses the full execution JSON to build a structured array of error objects with metadata and node-specific error details.  
    - *Configuration:* Custom JS code that:  
      - Extracts execution metadata (IDs, timestamps, workflow info)  
      - Identifies the trigger node and its payload  
      - Iterates over all node runs, excluding nodes with "SERP" in their names  
      - Collects error details including error name, message, stack, parameters, credentials used  
    - *Connections:* Input from Remove Manual Exec, output to Error Solver Agent.  
    - *Edge Cases:* Execution data may be incomplete or missing fields; JSON parsing errors; nodes without errors yield empty results; no errors found returns a success message.  
    - *Version:* Requires n8n supporting JavaScript code nodes with ES6 features.

---

#### 2.3 AI-Powered Error Diagnosis

- **Overview:**  
  Sends the extracted error details to OpenAI’s GPT-4o model wrapped in a LangChain agent node to generate a professional diagnosis, cause, and resolution for the error.

- **Nodes Involved:**  
  - OpenAI Chat Model  
  - Structured Output Parser  
  - Error Solver Agent  

- **Node Details:**

  - **OpenAI Chat Model**  
    - *Type:* @n8n/n8n-nodes-langchain.lmChatOpenAi  
    - *Role:* Provides the GPT-4o language model interface for generating AI responses.  
    - *Configuration:* Model set to "gpt-4o" (OpenAI GPT-4o variant), no additional options set.  
    - *Credentials:* Requires OpenAI API key credentials.  
    - *Connections:* Outputs to Error Solver Agent’s language model input.  
    - *Edge Cases:* API rate limits, invalid credentials, network errors, model unavailability.

  - **Structured Output Parser**  
    - *Type:* @n8n/n8n-nodes-langchain.outputParserStructured  
    - *Role:* Defines expected JSON schema for AI output with fields diagnosis, cause, and resolution.  
    - *Configuration:* JSON schema example provided to enforce structured AI responses.  
    - *Connections:* Outputs to Error Solver Agent’s output parser input.  
    - *Edge Cases:* AI response not matching schema may cause parsing errors.

  - **Error Solver Agent**  
    - *Type:* @n8n/n8n-nodes-langchain.agent  
    - *Role:* Orchestrates the AI prompt and response parsing.  
    - *Configuration:*  
      - System message sets the AI persona as a seasoned n8n expert.  
      - Prompt includes the full error JSON stringified for context.  
      - Output parser enabled to parse structured JSON output.  
    - *Connections:* Input from Extract Error Details, outputs to Set Diagnosis Fields.  
    - *Edge Cases:* Large error JSON may exceed token limits; malformed JSON input; AI hallucination or irrelevant output.

---

#### 2.4 Formatting & Email Reporting

- **Overview:**  
  Formats the AI-generated diagnosis into a styled HTML email and sends it to the configured recipients via Gmail SMTP.

- **Nodes Involved:**  
  - Set Diagnosis Fields  
  - Generate Email  
  - Send Gmail  

- **Node Details:**

  - **Set Diagnosis Fields**  
    - *Type:* n8n-nodes-base.set  
    - *Role:* Maps AI output fields (diagnosis, cause, resolution) and execution metadata into a structured JSON object for email generation.  
    - *Configuration:* Assigns multiple fields including workflow name, execution ID, links to workflow and execution, timestamps, node names, and raw JSON string.  
    - *Connections:* Input from Error Solver Agent, output to Generate Email.  
    - *Edge Cases:* Missing AI output fields or execution metadata may cause incomplete emails.

  - **Generate Email**  
    - *Type:* n8n-nodes-base.code (JavaScript code node)  
    - *Role:* Builds a fully styled HTML email body incorporating error details, diagnosis, cause, and resolution.  
    - *Configuration:*  
      - Parses input JSON array of errors.  
      - Generates HTML blocks with inline CSS for each error.  
      - Includes links to workflow and execution in n8n UI.  
      - Adds branding and footer with external links.  
    - *Connections:* Input from Set Diagnosis Fields, output to Send Gmail.  
    - *Edge Cases:* HTML injection risks if input not sanitized; date formatting depends on timezone; empty error array handled gracefully.

  - **Send Gmail**  
    - *Type:* n8n-nodes-base.gmail  
    - *Role:* Sends the generated HTML email to recipients.  
    - *Configuration:*  
      - Uses TO, CC, BCC from SET EMAIL node.  
      - Subject line includes workflow name and execution ID.  
      - Appends attribution automatically.  
    - *Credentials:* Requires Gmail OAuth2 credentials.  
    - *Connections:* Input from Generate Email.  
    - *Edge Cases:* Authentication errors, quota limits, invalid email addresses, network issues.

---

### 3. Summary Table

| Node Name           | Node Type                              | Functional Role                          | Input Node(s)          | Output Node(s)         | Sticky Note                                                                                   |
|---------------------|--------------------------------------|----------------------------------------|------------------------|------------------------|-----------------------------------------------------------------------------------------------|
| Error Trigger       | n8n-nodes-base.errorTrigger           | Triggers workflow on any n8n error     | —                      | SET EMAIL              | Automated Error Reporter with AI-Powered Diagnosis: Captures any n8n error, sends to OpenAI, and emails report. Requires OpenAI and SMTP credentials. |
| SET EMAIL           | n8n-nodes-base.set                    | Defines email recipients (TO, CC, BCC) | Error Trigger          | Get Failed Exec        | # SET YOUR EMAILS                                                                             |
| Get Failed Exec     | n8n-nodes-base.n8n (API node)        | Fetches full failed execution data     | SET EMAIL              | Remove Manual Exec      |                                                                                               |
| Remove Manual Exec  | n8n-nodes-base.if                    | Filters out manual executions          | Get Failed Exec        | Extract Error Details   | # Enable/Disable Manual Executions                                                            |
| Extract Error Details| n8n-nodes-base.code                  | Extracts detailed error metadata       | Remove Manual Exec     | Error Solver Agent      |                                                                                               |
| OpenAI Chat Model   | @n8n/n8n-nodes-langchain.lmChatOpenAi| Provides GPT-4o AI model interface     | Error Solver Agent     | Error Solver Agent      |                                                                                               |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI output into structured JSON  | Error Solver Agent     | Error Solver Agent      |                                                                                               |
| Error Solver Agent  | @n8n/n8n-nodes-langchain.agent       | Orchestrates AI prompt and parsing     | Extract Error Details  | Set Diagnosis Fields    |                                                                                               |
| Set Diagnosis Fields| n8n-nodes-base.set                    | Maps AI output and metadata for email  | Error Solver Agent     | Generate Email          |                                                                                               |
| Generate Email      | n8n-nodes-base.code                  | Builds styled HTML email body          | Set Diagnosis Fields   | Send Gmail              |                                                                                               |
| Send Gmail          | n8n-nodes-base.gmail                 | Sends the email report                  | Generate Email         | —                      |                                                                                               |
| Sticky Note         | n8n-nodes-base.stickyNote            | Visual note: # SET YOUR EMAILS          | —                      | —                      | # SET YOUR EMAILS                                                                             |
| Sticky Note1        | n8n-nodes-base.stickyNote            | Visual note: # Enable/Disable Manual Executions | —                      | —                      | # Enable/Disable Manual Executions                                                            |

---

### 4. Reproducing the Workflow from Scratch

1. **Create the "Error Trigger" node**  
   - Type: Error Trigger  
   - Configuration: Default, triggers on any workflow error globally.  
   - Position: Start of the workflow.

2. **Create the "SET EMAIL" node**  
   - Type: Set  
   - Configuration: Assign three string fields:  
     - TO: your admin email (e.g., myemail@myemail.com)  
     - CC: optional CC email(s)  
     - BCC: optional BCC email(s)  
   - Connect Error Trigger output to SET EMAIL input.

3. **Create the "Get Failed Exec" node**  
   - Type: n8n API node (resource: execution, operation: get)  
   - Configuration:  
     - Execution ID: `={{ $('Error Trigger').item.json.execution.id }}`  
     - Active workflows only: true  
   - Credentials: Configure with valid n8n API credentials.  
   - Connect SET EMAIL output to Get Failed Exec input.

4. **Create the "Remove Manual Exec" node**  
   - Type: If node  
   - Configuration:  
     - Condition: Execution mode string does NOT contain "manual" (case-insensitive)  
     - Use version 2.2 or later for advanced string operations.  
   - Connect Get Failed Exec output to Remove Manual Exec input.

5. **Create the "Extract Error Details" node**  
   - Type: Code (JavaScript)  
   - Configuration: Paste the provided JS code that:  
     - Extracts execution metadata (IDs, timestamps, workflow info)  
     - Identifies trigger node and payload  
     - Iterates over all nodes excluding those with "SERP" in their names  
     - Collects error details and returns an array of error objects or a success message if none found  
   - Connect Remove Manual Exec output (true branch) to Extract Error Details input.

6. **Create the "OpenAI Chat Model" node**  
   - Type: LangChain OpenAI Chat Model  
   - Configuration:  
     - Model: gpt-4o  
     - No additional options needed.  
   - Credentials: Add OpenAI API key credentials.  
   - Connect Extract Error Details output to OpenAI Chat Model input.

7. **Create the "Structured Output Parser" node**  
   - Type: LangChain Structured Output Parser  
   - Configuration: Provide JSON schema example:  
     ```json
     {
       "diagnosis": "",
       "cause": "",
       "resolution": ""
     }
     ```  
   - Connect Structured Output Parser output to OpenAI Chat Model’s output parser input.

8. **Create the "Error Solver Agent" node**  
   - Type: LangChain Agent  
   - Configuration:  
     - Prompt type: Define  
     - System message:  
       "You are an seasoned n8n expert with specializations in managing n8n instances and workflows. The user will provide a detailed error json object and your goal is to review, analyze and understand the error and using your expertise diagnose the error and provide a detailed report to the user with your diagnosis, cause and resolution so the user understands and can immediately fix the issue."  
     - Text input:  
       `=Can you please help me with this error that occured in my n8n workflow? {{ JSON.stringify($json) }}`  
     - Enable output parser.  
   - Connect Extract Error Details output to Error Solver Agent input.  
   - Connect OpenAI Chat Model output to Error Solver Agent language model input.  
   - Connect Structured Output Parser output to Error Solver Agent output parser input.

9. **Create the "Set Diagnosis Fields" node**  
   - Type: Set  
   - Configuration: Assign multiple fields mapping AI output and execution metadata, for example:  
     - error.diagnosis = `={{ $json.output.diagnosis }}`  
     - error.cause = `={{ $json.output.cause }}`  
     - error.resolution = `={{ $json.output.resolution }}`  
     - error.workflowName = `={{ $('Extract Error Details').item.json.workflowName }}`  
     - error.executionId = `={{ $('Extract Error Details').item.json.executionId }}`  
     - workflowLink = `={{ $execution.resumeUrl.split('/').slice(0, 3).join('/') }}/workflow/{{ $('Extract Error Details').item.json.workflowId }}`  
     - executionLink = `={{ $execution.resumeUrl.split('/').slice(0, 3).join('/') }}/workflow/{{ $('Extract Error Details').item.json.workflowId }}/executions/{{ $('Extract Error Details').item.json.executionId }}`  
     - Additional fields as per the original node.  
   - Connect Error Solver Agent output to Set Diagnosis Fields input.

10. **Create the "Generate Email" node**  
    - Type: Code (JavaScript)  
    - Configuration: Paste the provided JS code that:  
      - Parses the error array input  
      - Generates styled HTML email content with error details, diagnosis, cause, resolution, and links  
      - Sets email subject line with workflow name and execution ID  
    - Connect Set Diagnosis Fields output to Generate Email input.

11. **Create the "Send Gmail" node**  
    - Type: Gmail  
    - Configuration:  
      - Send To: `={{ $('SET EMAIL').item.json.TO }}`  
      - CC: `={{ $('SET EMAIL').item.json.CC }}`  
      - BCC: `={{ $('SET EMAIL').item.json.BCC }}`  
      - Subject: `={{ $json.subject }}`  
      - Message: `={{ $json.html }}`  
      - Append attribution: enabled  
    - Credentials: Configure Gmail OAuth2 credentials.  
    - Connect Generate Email output to Send Gmail input.

12. **Add Sticky Notes** (optional for clarity)  
    - Add a sticky note near SET EMAIL node with content: "# SET YOUR EMAILS"  
    - Add a sticky note near Remove Manual Exec node with content: "# Enable/Disable Manual Executions"

13. **Activate the workflow** and test by triggering an error in any workflow to verify email delivery and AI diagnosis.

---

### 5. General Notes & Resources

| Note Content                                                                                                            | Context or Link                                                                                  |
|-------------------------------------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------|
| This workflow requires the OpenAI node enabled on your n8n instance (Cloud or self-hosted).                             | Setup instructions in the workflow description.                                                |
| SMTP credentials (Gmail OAuth2 recommended) must be configured for email sending.                                        | Gmail OAuth2 credentials setup in n8n.                                                         |
| The HTML email includes branding and links to Real Simple Solutions and a Gumroad collection of n8n AI automation templates. | https://realsimple.dev and https://joeper.es/4jXyRub                                           |
| The workflow excludes nodes with "SERP" in their name from error extraction to avoid noise from specific integrations.   | Custom logic in Extract Error Details node.                                                    |
| Timezone for date formatting in emails is set to America/New_York; adjust in Generate Email node as needed.              | JavaScript Date formatting in Generate Email node.                                            |

---

This documentation provides a complete, structured understanding of the FloWatch workflow, enabling reproduction, modification, and troubleshooting by advanced users or automated systems.