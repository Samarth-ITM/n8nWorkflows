Resume Data Extraction and Storage in Supabase from Email Attachments

https://n8nworkflows.xyz/workflows/resume-data-extraction-and-storage-in-supabase-from-email-attachments-4106


# Resume Data Extraction and Storage in Supabase from Email Attachments

### 1. Workflow Overview

This workflow automates the extraction of structured candidate information from resumes received as email attachments and stores the data in a Supabase database. It is designed for HR professionals and recruiters to streamline resume processing by removing manual data entry.

The workflow is logically divided into four main blocks:

- **1.1 Email Monitoring and Attachment Check:** Watches a Gmail inbox for new emails with attachments, filtering to ensure resumes are present before processing.
- **1.2 Data Preparation:** Extracts the content of the attached resume files (PDF format) for further analysis.
- **1.3 AI-Powered Information Extraction:** Uses an AI language model (via OpenRouter) to parse the extracted resume text into structured fields such as Name, Email, Telephone, Education, Experience, and Skills.
- **1.4 Data Storage:** Inserts the extracted and structured data into multiple Supabase database tables, including main candidate info, skills, experience, and education.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Monitoring and Attachment Check

- **Overview:**  
  This block triggers the workflow when a new email arrives in the monitored Gmail inbox. It verifies that the email contains at least one attachment to proceed with resume processing.

- **Nodes Involved:**  
  - Gmail Trigger  
  - If  
  - Sticky Note (Email Trigger)  
  - Sticky Note (check if Attachment exists)

- **Node Details:**

  - **Gmail Trigger**  
    - Type: Trigger node for Gmail API  
    - Configuration: Polls every minute, downloads attachments automatically  
    - Inputs: None (trigger)  
    - Outputs: New email data including attachments  
    - Failure Modes: Auth errors on Gmail API, quota limits, no attachments in email  
    - Notes: Requires Gmail API credentials with appropriate scopes.

  - **If**  
    - Type: Conditional node  
    - Configuration: Checks if the email payload contains an attachment property named "attachment_0"  
    - Inputs: Output from Gmail Trigger  
    - Outputs: Proceeds only if attachment exists  
    - Failure Modes: Expression evaluation errors if attachment property is missing or malformed

  - **Sticky Notes**  
    - Provide high-level context: "Email Trigger" and "check if Attachment exists"  
    - No functional role

#### 2.2 Data Preparation

- **Overview:**  
  This block extracts the textual content from the attached resume file (PDF format) and prepares it for AI parsing.

- **Nodes Involved:**  
  - Extract from File  
  - Edit Fields  
  - Sticky Note (Prepare Data)

- **Node Details:**

  - **Extract from File**  
    - Type: File extraction node  
    - Configuration: Extracts text from the binary property "attachment_0" assuming PDF format  
    - Inputs: Attachment binary data from If node  
    - Outputs: Extracted raw text of resume as JSON property `text`  
    - Failure Modes: Unsupported file type, corrupted PDF, missing binary data

  - **Edit Fields**  
    - Type: Set node  
    - Configuration: Assigns extracted text to a field named `text` explicitly for consistent downstream referencing  
    - Inputs: Extracted text from previous node  
    - Outputs: JSON with `text` field containing resume text  
    - Failure Modes: Expression errors if input JSON structure varies

  - **Sticky Note**  
    - Label: "Prepare Data"  
    - No functional role

#### 2.3 AI-Powered Information Extraction

- **Overview:**  
  Uses an AI language model powered by OpenRouter to parse the resume text into structured fields: Name, Email, Telephone, Experience, Skills, Education.

- **Nodes Involved:**  
  - Basic LLM Chain  
  - OpenRouter Chat Model  
  - Structured Output Parser  
  - Sticky Note (Extract Information)

- **Node Details:**

  - **Basic LLM Chain**  
    - Type: Langchain LLM Chain node  
    - Configuration: Prompt instructs AI as an HR expert to extract specific fields from given resume text (`text` field)  
    - Inputs: Prepared resume text  
    - Outputs: Structured JSON with extracted fields  
    - Dependencies: Uses OpenRouter Chat Model as language model and Structured Output Parser for JSON output  
    - Failure Modes: API key issues for OpenRouter, prompt failures, parsing errors, rate limits

  - **OpenRouter Chat Model**  
    - Type: AI language model node  
    - Configuration: Uses "meta-llama/llama-4-scout:free" model on OpenRouter platform  
    - Inputs: Prompt from Basic LLM Chain  
    - Outputs: AI-generated response for parsing  
    - Failure Modes: Model availability, network errors, auth issues

  - **Structured Output Parser**  
    - Type: Output parser for LLM  
    - Configuration: Enforces expected JSON schema with example fields for validation  
    - Inputs: AI model raw output  
    - Outputs: Parsed structured JSON fields (Name, Telephone, Email, Education, Skill, Experience)  
    - Failure Modes: Schema mismatch, invalid JSON from AI, parsing exceptions

  - **Sticky Note**  
    - Label: "Extract Information"  
    - No functional role

#### 2.4 Data Storage

- **Overview:**  
  Inserts the extracted structured data into multiple Supabase tables: a main table for candidate overview and separate tables for experience, skills, and education entries.

- **Nodes Involved:**  
  - HTTP Request  
  - Edit Fields1, Edit Fields2, Edit Fields3  
  - Split Out, Split Out1, Split Out2  
  - Supabase, Supabase1, Supabase2  
  - Sticky Note (Save Information)

- **Node Details:**

  - **HTTP Request**  
    - Type: HTTP Request node  
    - Configuration: POSTs main candidate info (name, telephone, email) to Supabase REST endpoint; headers include `apikey` and `Authorization` with bearer token  
    - Inputs: Output from Basic LLM Chain with structured data  
    - Outputs: Supabase response containing inserted record ID  
    - Failure Modes: Auth errors, invalid URL, network issues, data validation errors

  - **Edit Fields1 (Experience), Edit Fields2 (Skills), Edit Fields3 (Education)**  
    - Type: Set nodes  
    - Configuration: Extract respective arrays (Experience, Skill, Education) from AI output into fields `Expr`, `skill`, and `Educ`  
    - Inputs: Supabase HTTP response and AI output  
    - Outputs: Arrays of entries for further splitting  
    - Failure Modes: Missing or empty arrays, expression errors

  - **Split Out, Split Out1, Split Out2**  
    - Type: Split Out nodes  
    - Configuration: Decompose arrays of Experience, Skills, and Education into individual items for separate database entries  
    - Inputs: Corresponding Edit Fields nodes  
    - Outputs: Individual records for insertion  
    - Failure Modes: Empty arrays, malformed entries

  - **Supabase, Supabase1, Supabase2**  
    - Type: Supabase nodes  
    - Configuration: Insert each individual record into respective Supabase tables:  
      - `Supabase`: table `experiences` with fields `cv_id` (from Supabase insert ID) and `text` (experience text)  
      - `Supabase1`: table `skills` with `cv_id` and `text` (skill)  
      - `Supabase2`: table `education` with `cv_id` and `text` (education entry)  
    - Inputs: Split Out nodes  
    - Outputs: Insert operation results  
    - Failure Modes: Auth errors, data validation failures, network issues

  - **Sticky Note**  
    - Label: "Save Information"  
    - No functional role

---

### 3. Summary Table

| Node Name            | Node Type                      | Functional Role                      | Input Node(s)          | Output Node(s)                        | Sticky Note                      |
|----------------------|--------------------------------|------------------------------------|-----------------------|-------------------------------------|---------------------------------|
| Gmail Trigger        | n8n-nodes-base.gmailTrigger    | Email inbox monitoring trigger     | None                  | If                                  | ## Email Trigger                |
| If                   | n8n-nodes-base.if              | Checks for presence of attachment  | Gmail Trigger         | Extract from File                    | ## check if Attachment exists  |
| Extract from File     | n8n-nodes-base.extractFromFile | Extracts text from PDF attachment  | If                    | Edit Fields                        | ## Prepare Data                |
| Edit Fields          | n8n-nodes-base.set             | Prepares extracted text field      | Extract from File      | Basic LLM Chain                     | ## Prepare Data                |
| Basic LLM Chain       | @n8n/n8n-nodes-langchain.chainLlm | AI parsing of resume text to structured data | Edit Fields          | HTTP Request                       | ## Extract Information         |
| OpenRouter Chat Model | @n8n/n8n-nodes-langchain.lmChatOpenRouter | Provides AI language model service | Basic LLM Chain (ai_languageModel) | Basic LLM Chain                   | ## Extract Information         |
| Structured Output Parser | @n8n/n8n-nodes-langchain.outputParserStructured | Parses AI output into JSON schema | OpenRouter Chat Model (ai_outputParser) | Basic LLM Chain                 | ## Extract Information         |
| HTTP Request          | n8n-nodes-base.httpRequest     | Inserts main candidate data to Supabase | Basic LLM Chain       | Edit Fields1, Edit Fields2, Edit Fields3 | ## Save Information            |
| Edit Fields1          | n8n-nodes-base.set             | Prepares Experience array for insertion | HTTP Request          | Split Out                          | ## Save Information            |
| Split Out             | n8n-nodes-base.splitOut        | Splits Experience array into items | Edit Fields1           | Supabase                          | ## Save Information            |
| Supabase              | n8n-nodes-base.supabase        | Inserts individual experience items | Split Out             | None                              | ## Save Information            |
| Edit Fields2          | n8n-nodes-base.set             | Prepares Skill array for insertion  | HTTP Request           | Split Out1                         | ## Save Information            |
| Split Out1            | n8n-nodes-base.splitOut        | Splits Skill array into items       | Edit Fields2           | Supabase1                         | ## Save Information            |
| Supabase1             | n8n-nodes-base.supabase        | Inserts individual skill items      | Split Out1             | None                              | ## Save Information            |
| Edit Fields3          | n8n-nodes-base.set             | Prepares Education array for insertion | HTTP Request          | Split Out2                        | ## Save Information            |
| Split Out2            | n8n-nodes-base.splitOut        | Splits Education array into items   | Edit Fields3           | Supabase2                         | ## Save Information            |
| Supabase2             | n8n-nodes-base.supabase        | Inserts individual education items  | Split Out2             | None                              | ## Save Information            |
| Sticky Note           | n8n-nodes-base.stickyNote      | Visual label                       | None                  | None                              | ## Email Trigger               |
| Sticky Note1          | n8n-nodes-base.stickyNote      | Visual label                       | None                  | None                              | ## Prepare Data               |
| Sticky Note2          | n8n-nodes-base.stickyNote      | Visual label                       | None                  | None                              | ## Extract Information        |
| Sticky Note3          | n8n-nodes-base.stickyNote      | Visual label                       | None                  | None                              | ## Save Information           |
| Sticky Note4          | n8n-nodes-base.stickyNote      | Visual label                       | None                  | None                              | ## check if Attachment exists |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Gmail Trigger Node**  
   - Type: Gmail Trigger  
   - Set "Simple" to false  
   - Enable "Download Attachments" option  
   - Configure polling interval to every minute  
   - Attach Gmail API credentials (Client ID, Client Secret)  

2. **Create If Node to Check Attachment**  
   - Type: If  
   - Condition: Check if property `attachment_0` exists in the incoming email JSON  
   - Connect Gmail Trigger output to If input  

3. **Create Extract from File Node**  
   - Type: Extract from File  
   - Operation: PDF text extraction  
   - Binary Property Name: `attachment_0`  
   - Connect If's true output to this node  

4. **Create Edit Fields Node (Prepare Data)**  
   - Type: Set  
   - Add a string field `text` assigned to `={{ $json.text }}` from previous node  
   - Connect Extract from File output to this node  

5. **Create OpenRouter Chat Model Node**  
   - Type: Langchain Chat Model (OpenRouter)  
   - Model: "meta-llama/llama-4-scout:free"  
   - Attach OpenRouter API credentials  

6. **Create Structured Output Parser Node**  
   - Type: Langchain Output Parser Structured  
   - Provide JSON schema example with fields: Name, Telephone, Email, Education (array), Skill (array), Experience (array)  

7. **Create Basic LLM Chain Node**  
   - Type: Langchain Chain LLM  
   - Prompt:  
     ```
     You are an HR expert, you are given a detailed CV text 
     extract 
     Name 
     Email 
     Telephone 
     Experience 
     Skills 
     Education 
     This is the CV text: {{ $json.text }}
     ```  
   - Enable Output Parser and link it to Structured Output Parser  
   - Set AI language model to OpenRouter Chat Model  
   - Connect Edit Fields output to Basic LLM Chain input  

8. **Create HTTP Request Node for Supabase Insert**  
   - Method: POST  
   - URL: Your Supabase REST endpoint for main resume table  
   - Headers:  
     - `apikey`: Supabase anon/public API key  
     - `Authorization`: Bearer token with same key  
     - `Prefer`: `return=representation` (to get inserted record)  
   - Body Parameters:  
     - `name`: `={{ $json.output.Name }}`  
     - `telephone`: `={{ $json.output.Telephone }}`  
     - `email`: `={{ $json.output.Email }}`  
   - Connect Basic LLM Chain output to this node  

9. **Create Edit Fields Nodes for Arrays**  
   - Edit Fields1: field `Expr` = `={{ $('Basic LLM Chain').item.json.output.Experience }}`  
   - Edit Fields2: field `skill` = `={{ $('Basic LLM Chain').item.json.output.Skill }}`  
   - Edit Fields3: field `Educ` = `={{ $('Basic LLM Chain').item.json.output.Education }}`  
   - Connect HTTP Request output to all three Edit Fields nodes  

10. **Create Split Out Nodes for Arrays**  
    - Split Out: fieldToSplitOut = `Expr`  
    - Split Out1: fieldToSplitOut = `skill`  
    - Split Out2: fieldToSplitOut = `Educ`  
    - Connect Edit Fields1 to Split Out, Edit Fields2 to Split Out1, Edit Fields3 to Split Out2  

11. **Create Supabase Nodes to Insert Array Items**  
    - Supabase (Experiences table): fields `cv_id` = inserted record id from HTTP Request, `text` = `={{ $json.Expr }}`  
    - Supabase1 (Skills table): fields `cv_id` and `text` = `={{ $json.skill }}`  
    - Supabase2 (Education table): fields `cv_id` and `text` = `={{ $json.Educ }}`  
    - Connect Split Out to Supabase, Split Out1 to Supabase1, Split Out2 to Supabase2  
    - Attach Supabase credentials with appropriate rights  

12. **Add Sticky Notes for Documentation**  
    - Place sticky notes around logical blocks to label: "Email Trigger", "check if Attachment exists", "Prepare Data", "Extract Information", and "Save Information"  

13. **Save and Activate Workflow**  
    - Test workflow by sending an email with a resume attachment  
    - Monitor execution and verify data in Supabase tables  

---

### 5. General Notes & Resources

| Note Content                                                                                                 | Context or Link                                                                                   |
|--------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| Workflow automates resume data extraction and storage to streamline hiring processes for HR professionals. | Workflow Description in imported n8n workflow metadata                                          |
| Uses OpenRouter platform for AI language model management, centralizing API keys.                            | https://openrouter.ai/ - for API key management and AI model access                              |
| Supabase database must be pre-configured with tables: experiences, skills, education matching workflow fields.| Supabase documentation: https://supabase.com/docs                                               |
| Gmail API credentials require OAuth2 setup in Google Cloud Platform with Gmail API enabled.                  | Google Cloud Console: https://console.cloud.google.com/apis/library/gmail.googleapis.com         |
| Ensure file attachments are PDF format for proper extraction; other formats require node adaptations.         | Extract from File node supports PDF extraction by default                                       |

---

**Disclaimer:** The text provided stems exclusively from an automated workflow created with n8n, respecting all content policies and handling only legal, public data.