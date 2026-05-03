Classify Emails & Extract Structured Data from Job Applications with GPT-4o

https://n8nworkflows.xyz/workflows/classify-emails---extract-structured-data-from-job-applications-with-gpt-4o-3457


# Classify Emails & Extract Structured Data from Job Applications with GPT-4o

### 1. Workflow Overview

This workflow automates the classification and structured data extraction from incoming emails, specifically targeting job applications. It is designed for business owners and HR professionals who want to streamline processing of unstructured job application emails by categorizing them and extracting key applicant details automatically using OpenAI GPT-4o models.

The workflow consists of the following logical blocks:

- **1.1 Email Reception and Attachment Extraction:** Connects to an email inbox via IMAP, triggers on new emails, and extracts textual content from PDF attachments.

- **1.2 Email Classification:** Uses an OpenAI GPT-4o-based text classifier to categorize the incoming email and its attachment content into predefined categories such as job applications, inbound leads, invoices, or others.

- **1.3 Structured Data Extraction:** For emails classified as job applications, another GPT-4o model extracts structured applicant details (e.g., name, age, education) from the email and attachment texts.

- **1.4 Branching to Category-Specific Workflows:** Based on classification, directs flow to further workflows or processing steps tailored to each email category.

This design allows easy extension by adding or modifying categories and their corresponding workflows, facilitating automation of various email processing scenarios beyond job applications.

---

### 2. Block-by-Block Analysis

#### 2.1 Email Reception and Attachment Extraction

- **Overview:**  
  This block listens for new incoming emails via an IMAP account, downloads the email content, and extracts text from PDF attachments to supplement classification and extraction steps.

- **Nodes Involved:**  
  - Email trigger  
  - Extract data from attachment

- **Node Details:**

  - **Email trigger**  
    - Type: `n8n-nodes-base.emailReadImap`  
    - Technical role: Listens for new emails on a configured IMAP account and outputs email data with resolved content and attachments.  
    - Configuration:  
      - Format set to "resolved" to get structured text and attachments.  
      - No post-processing action configured (set to "nothing").  
      - Prefix for attachment binary data set as "attachment".  
      - Uses preconfigured IMAP credentials to connect to the email server.  
    - Inputs: None (trigger node)  
    - Outputs: Email data including plain text and binary attachments.  
    - Edge cases and failures:  
      - IMAP authentication failure or connectivity issues.  
      - Emails without attachments or with unsupported attachment formats.  
      - Attachment extraction may fail if the attachment is not a PDF or is corrupted.

  - **Extract data from attachment**  
    - Type: `n8n-nodes-base.extractFromFile`  
    - Technical role: Extracts text from the first email attachment (assumed PDF).  
    - Configuration:  
      - Operation set to "pdf" to extract text from PDF files.  
      - Binary property name set to "attachment0" referencing the first attachment from the email trigger.  
      - On error set to "continueRegularOutput" to avoid workflow stoppage on extraction failure.  
    - Inputs: Receives email data from "Email trigger" node.  
    - Outputs: Extracted text from the PDF attachment.  
    - Edge cases and failures:  
      - Non-PDF attachments or encrypted PDFs will cause extraction failure (handled gracefully).  
      - Missing or empty attachments result in no text extracted.

---

#### 2.2 Email Classification

- **Overview:**  
  This block classifies the incoming email (including extracted attachment text) into one of several predefined categories using an OpenAI GPT-4o-powered text classifier.

- **Nodes Involved:**  
  - Classify email  
  - OpenAI Chat Model

- **Node Details:**

  - **Classify email**  
    - Type: `@n8n/n8n-nodes-langchain.textClassifier`  
    - Technical role: Uses an AI-powered classifier to assign the email to a category based on analyzed content.  
    - Configuration:  
      - Input text combines the plain email text (`Email trigger`) and extracted attachment text (`Extract data from attachment`) via expressions.  
      - Categories defined:  
        - `job_application`: for job applications  
        - `inbound_lead`: for sales inquiries  
        - `invoice`: for invoices  
        - `other`: for all other emails  
      - Each category has a clear description to guide classification.  
    - Inputs: Receives extracted texts from "Email trigger" and "Extract data from attachment" nodes.  
    - Outputs: Classification result with assigned category.  
    - Edge cases and failures:  
      - Model misclassification if email content is ambiguous or sparse.  
      - Model may fail or timeout if OpenAI API is unreachable or API key invalid.  
      - Expression errors if referenced nodes do not provide expected data.  
    - Version-specific: Requires n8n version supporting `@n8n/n8n-nodes-langchain.textClassifier` and GPT-4o model compatibility.

  - **OpenAI Chat Model**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Technical role: Provides GPT-4o model backend for the classifier node.  
    - Configuration:  
      - Model set to "gpt-4o".  
      - Connected to OpenAI API credentials.  
    - Inputs: Receives classification prompts from "Classify email" node.  
    - Outputs: Classification inference to "Classify email".  
    - Edge cases and failures:  
      - OpenAI API key issues or rate limits.  
      - Network or API downtime.

---

#### 2.3 Structured Data Extraction

- **Overview:**  
  For emails classified as `job_application`, this block extracts structured candidate information such as name, age, education, and work experience from the email and attachment text.

- **Nodes Involved:**  
  - Extract variables - email & attachment  
  - OpenAI Chat Model 2

- **Node Details:**

  - **Extract variables - email & attachment**  
    - Type: `@n8n/n8n-nodes-langchain.informationExtractor`  
    - Technical role: Extracts multiple structured attributes from unstructured text using GPT-4o.  
    - Configuration:  
      - Input text includes plain email text and attachment-extracted resume text.  
      - Attributes to extract:  
        - first_name, last_name, age, residence, study, work_experience, personal_character  
      - Each attribute has a concise description to guide extraction accuracy.  
    - Inputs: Receives combined text data from "Email trigger" and "Extract data from attachment".  
    - Outputs: JSON with extracted structured fields.  
    - Edge cases and failures:  
      - Missing or incomplete data in email or attachment may lead to partial extraction.  
      - Model API failures or timeouts.  
      - Expression errors if input data missing.  
    - Sub-workflow reference: None.

  - **OpenAI Chat Model 2**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Technical role: GPT-4o model backend for the information extractor node.  
    - Configuration:  
      - Model set to "gpt-4o".  
      - Uses the same OpenAI API credentials as the classifier.  
    - Inputs: Receives extraction prompts from "Extract variables - email & attachment".  
    - Outputs: Extracted structured data.  
    - Edge cases: Same OpenAI API failure risks as above.

---

#### 2.4 Branching to Category-Specific Workflows

- **Overview:**  
  After classification, the workflow routes execution to different no-operation (placeholder) nodes representing category-specific follow-up workflows. These can be replaced or extended to integrate with CRM/ATS systems or other processes.

- **Nodes Involved:**  
  - Workflow 2 (job_application branch)  
  - Workflow 3 (inbound_lead branch)  
  - workflow 4 (invoice branch)  
  - (No explicit node for "other" category; can be added similarly.)

- **Node Details:**

  - **Workflow 2**  
    - Type: `n8n-nodes-base.noOp`  
    - Role: Placeholder node for processing classified job application emails (branch 1).  
    - Inputs: From "Classify email" node's main output index 1.  
    - Outputs: None  
    - Notes: Replace with actual workflow logic for job application handling (e.g., CRM integration).

  - **Workflow 3**  
    - Type: `n8n-nodes-base.noOp`  
    - Role: Placeholder for inbound lead processing workflows (branch 2).  
    - Inputs: From "Classify email" node's main output index 2.  
    - Outputs: None

  - **workflow 4**  
    - Type: `n8n-nodes-base.noOp`  
    - Role: Placeholder for invoice processing workflows (branch 3).  
    - Inputs: From "Classify email" node's main output index 3.  
    - Outputs: None

---

### 3. Summary Table

| Node Name                      | Node Type                                     | Functional Role                                | Input Node(s)               | Output Node(s)                          | Sticky Note                                                                                  |
|--------------------------------|-----------------------------------------------|-----------------------------------------------|-----------------------------|---------------------------------------|----------------------------------------------------------------------------------------------|
| Email trigger                  | n8n-nodes-base.emailReadImap                   | Email reception trigger via IMAP               | None                        | Extract data from attachment          |                                                                                              |
| Extract data from attachment   | n8n-nodes-base.extractFromFile                  | Extract text from PDF attachment                | Email trigger               | Classify email                        |                                                                                              |
| Classify email                | @n8n/n8n-nodes-langchain.textClassifier          | Classify email + attachment text into category | Extract data from attachment | Extract variables - email & attachment, Workflow 2, Workflow 3, workflow 4 | ### Change or add any category you want<br>Each category can be assigned it's own specific workflow |
| Extract variables - email & attachment | @n8n/n8n-nodes-langchain.informationExtractor | Extract structured applicant data from texts   | Classify email              | None                                  |                                                                                              |
| OpenAI Chat Model             | @n8n/n8n-nodes-langchain.lmChatOpenAi           | GPT-4o backend for classification               | Classify email (ai_languageModel) | Classify email                     |                                                                                              |
| OpenAI Chat Model 2           | @n8n/n8n-nodes-langchain.lmChatOpenAi           | GPT-4o backend for structured data extraction   | Extract variables - email & attachment (ai_languageModel) | Extract variables - email & attachment |                                                                                              |
| Workflow 2                   | n8n-nodes-base.noOp                              | Placeholder for job application workflow branch | Classify email (main output 1) | None                                  |                                                                                              |
| Workflow 3                   | n8n-nodes-base.noOp                              | Placeholder for inbound lead workflow branch    | Classify email (main output 2) | None                                  |                                                                                              |
| workflow 4                   | n8n-nodes-base.noOp                              | Placeholder for invoice workflow branch          | Classify email (main output 3) | None                                  |                                                                                              |
| Sticky Note                  | n8n-nodes-base.stickyNote                        | Instructional note                              | None                        | None                                  | ### Change or add any category you want<br>Each category can be assigned it's own specific workflow |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "Email trigger" node**  
   - Type: "EmailReadImap"  
   - Configure IMAP credentials for your email account (see [n8n IMAP docs](https://docs.n8n.io/integrations/builtin/credentials/imap/#related-resources))  
   - Set format to "resolved" to get parsed email content and attachments  
   - Leave "postProcessAction" as "nothing"  
   - Set "dataPropertyAttachmentsPrefixName" to "attachment"  
   - Position this node at the leftmost start of the workflow.

2. **Create "Extract data from attachment" node**  
   - Type: "ExtractFromFile"  
   - Set operation to "pdf" to extract text from PDF attachments  
   - Set binaryPropertyName to "attachment0" (first email attachment)  
   - Set "onError" to "continueRegularOutput" to allow workflow continuation if extraction fails  
   - Connect output of "Email trigger" node to input of this node.

3. **Create "OpenAI Chat Model" node** (GPT-4o for classification)  
   - Type: "@n8n/n8n-nodes-langchain.lmChatOpenAi"  
   - Configure with your OpenAI API credentials  
   - Set model to "gpt-4o"  
   - Position near "Classify email" node to serve as AI backend.

4. **Create "Classify email" node**  
   - Type: "@n8n/n8n-nodes-langchain.textClassifier"  
   - Input text:  
     ```n8n
     ={{ $('Email trigger').first().json.text }}

     attachment:
     {{ $('Extract data from attachment').first().json.text }}
     ```  
   - Define categories with clear descriptions:  
     - job_application: for job applications  
     - inbound_lead: for sales inquiries or info requests  
     - invoice: for invoices  
     - other: for all other emails  
   - Connect input from "Extract data from attachment" node output  
   - Connect "ai_languageModel" input to "OpenAI Chat Model" node output.

5. **Create "OpenAI Chat Model 2" node** (GPT-4o for extraction)  
   - Same setup as "OpenAI Chat Model" with same credentials and model "gpt-4o".

6. **Create "Extract variables - email & attachment" node**  
   - Type: "@n8n/n8n-nodes-langchain.informationExtractor"  
   - Input text:  
     ```n8n
     ={{ $('Email trigger').first().json.text }}

     Resume:
     {{ $('Extract data from attachment').first().json.text }}
     ```  
   - Define attributes to extract with descriptions:  
     - first_name: first name of the applicant  
     - last_name: last name of the applicant  
     - age: age of the applicant  
     - residence: residence of the applicant  
     - study: relevant completed study  
     - work_experience: relevant work experience  
     - personal_character: personal characteristics  
   - Connect input from "Classify email" node output (main output 0)  
   - Connect "ai_languageModel" input to "OpenAI Chat Model 2" output.

7. **Create placeholder nodes for branching workflows**  
   - Create three "NoOp" nodes named "Workflow 2", "Workflow 3", and "workflow 4"  
   - Connect outputs of "Classify email" node main outputs 1, 2, and 3 to these nodes respectively to represent email classification categories:  
     - Workflow 2: job_application  
     - Workflow 3: inbound_lead  
     - workflow 4: invoice  
   - These nodes can later be replaced with actual workflows for CRM integration or other processing.

8. **Add a "Sticky Note" node**  
   - Add a sticky note near the "Classify email" node with content:  
     ```
     ### Change or add any category you want  
     Each category can be assigned it's own specific workflow
     ```  
   - This serves as a reminder to customize categories and workflows.

9. **Set workflow settings**  
   - Optionally configure an error workflow for centralized error handling.  
   - Set execution order to "v1" (first-in-first-out).

10. **Activate the workflow**  
    - Test with incoming emails containing attachments.  
    - Monitor for API errors or extraction failures and adjust OpenAI/IMAP credentials as needed.

---

### 5. General Notes & Resources

| Note Content                                                                                                           | Context or Link                                                                                              |
|------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| Configure IMAP credentials for connecting your email account.                                                          | [n8n IMAP Credential Docs](https://docs.n8n.io/integrations/builtin/credentials/imap/#related-resources)    |
| Connect your OpenAI account in both "Classify email" and "Extract variables - email & attachment" nodes.               |                                                                                                             |
| Customize classification categories with clear descriptions to improve accuracy.                                       |                                                                                                             |
| Structured data extraction attributes can be adjusted to fit specific recruitment or CRM needs.                         |                                                                                                             |
| Consider integrating final workflow branches with CRM or ATS tools such as Hubspot or Recruitee for end-to-end handling.|                                                                                                             |

---

This documentation should provide a comprehensive understanding of the workflow’s structure, logic, and configuration, enabling users and AI agents to reproduce, maintain, and extend it confidently.