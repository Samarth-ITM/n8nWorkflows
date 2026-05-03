Automated Course Registration Evaluation with AI, Gmail, and Google Sheets

https://n8nworkflows.xyz/workflows/automated-course-registration-evaluation-with-ai--gmail--and-google-sheets-8567


# Automated Course Registration Evaluation with AI, Gmail, and Google Sheets

### 1. Workflow Overview

This workflow automates the evaluation and registration process for a course enrollment form using a combination of AI processing, Google services, and email notification. It is designed for a training program focused on API Testing & Automation. The workflow receives student submissions from a web form, evaluates if they meet course acceptance criteria using AI, stores the result and student data in Google Sheets and Pinecone vector index, and sends a personalized email notification about their enrollment status.

**Target Use Cases:**  
- Automating course registration decisions based on applicant responses  
- Logging enrollment data for record-keeping and analysis  
- Notifying students promptly about acceptance or rejection  
- Integrating AI to evaluate textual or categorical enrollment criteria  

**Logical Blocks:**  
- 1.1 Input Reception (Form Submission)  
- 1.2 User Data Preparation  
- 1.3 Course Enrollment Data Retrieval and Embedding  
- 1.4 AI Evaluation of Enrollment Eligibility  
- 1.5 Enrollment Status Notification Preparation  
- 1.6 Storage of Enrollment Records  
- 1.7 Email Notification Dispatch  

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception (Form Submission)  
**Overview:**  
This block triggers the workflow automatically when a student submits the enrollment form. It captures detailed registration data including name, email, phone, programming basics knowledge, agreement to terms, and commitment to the course schedule.

**Nodes Involved:**  
- On form submission  
- Sticky Note (documentation node)

**Node Details:**  
- **On form submission**  
  - Type: Form Trigger  
  - Role: Entry point for student registration data  
  - Configuration: Custom CSS styling for the form, multi-field form capturing personal details and commitments.  
  - Outputs: JSON object with all form fields on submission.  
  - Potential failures: Webhook connectivity issues, form validation errors, data missing required fields.  
- **Sticky Note**  
  - Type: Sticky Note (documentation)  
  - Role: Describes workflow start and purpose related to form submission.

---

#### 1.2 User Data Preparation  
**Overview:**  
Concatenates key user inputs into a single string for downstream AI processing and indexing.

**Nodes Involved:**  
- Combine User Data

**Node Details:**  
- **Combine User Data**  
  - Type: Set node  
  - Role: Creates a unified context string of user inputs (name, email, WhatsApp number, programming basics answer, terms agreement)  
  - Key expression: combines multiple form fields into a single string `user_context`  
  - Inputs: Form submission data  
  - Outputs: JSON with `user_context` property  
  - Potential failures: Expression errors if input fields missing; data formatting issues.

---

#### 1.3 Course Enrollment Data Retrieval and Embedding  
**Overview:**  
Downloads the master Google Sheet of course enrollment records, processes it into a vector store using Pinecone, and embeds textual data with Google Gemini embeddings for similarity search.

**Nodes Involved:**  
- Download file (Google Drive)  
- Default Data Loader  
- Embeddings Google Gemini  
- Pinecone Vector Store

**Node Details:**  
- **Download file**  
  - Type: Google Drive node  
  - Role: Downloads the Google Sheets file containing enrollment data  
  - Config: File ID of the Google Sheet, OAuth2 credentials for Google Drive  
  - Outputs: Binary file content (spreadsheet)  
  - Potential failures: Authentication errors, file access permission issues, file not found.  
- **Default Data Loader**  
  - Type: Document Loader  
  - Role: Converts the downloaded spreadsheet into documents for embedding  
  - Inputs: File content from Google Drive node  
- **Embeddings Google Gemini**  
  - Type: Embeddings Provider  
  - Role: Encodes loaded documents into embeddings using Google Gemini API  
  - Credentials: Google Palm API key  
- **Pinecone Vector Store**  
  - Type: Vector Store (Pinecone)  
  - Role: Inserts embeddings into Pinecone index under namespace "course_enrollment"  
  - Config: Pinecone index "course-enrollment", namespace set  
  - Credentials: Pinecone API key  
  - Potential failures: API rate limits, network errors, invalid API keys.

---

#### 1.4 AI Evaluation of Enrollment Eligibility  
**Overview:**  
Uses a LangChain AI agent with OpenAI GPT-4o-mini to evaluate student eligibility based on combined user data. The AI outputs a structured JSON indicating acceptance status.

**Nodes Involved:**  
- AI Agent  
- OpenAI Chat Model  
- Structured Output Parser

**Node Details:**  
- **AI Agent**  
  - Type: LangChain Agent  
  - Role: Defines the evaluation prompt instructing AI to assess acceptance based on conditions (basics = yes, terms agreed, commitment agreed)  
  - Input: `user_context` string from Combine User Data  
  - Output: JSON object with `accepted: true/false`  
- **OpenAI Chat Model**  
  - Type: Language Model (OpenAI GPT-4o-mini)  
  - Role: Processes AI prompt and returns raw model output  
  - Credentials: OpenAI API key  
- **Structured Output Parser**  
  - Type: Output Parser  
  - Role: Parses AI raw output into structured JSON format for downstream logic  
  - Config: JSON schema example with boolean `accepted` key  
  - Potential failures: Model API errors, parsing failures if AI output format unexpected.

---

#### 1.5 Enrollment Status Notification Preparation  
**Overview:**  
Prepares a second AI agent to craft a personalized email message to the student about their acceptance or rejection, referencing enrollment data and guiding rejected students to preparatory materials.

**Nodes Involved:**  
- AI Agent1  
- OpenAI Chat Model1

**Node Details:**  
- **AI Agent1**  
  - Type: LangChain Agent  
  - Role: Generates a professional email content (HTML template) tailored to acceptance or rejection status  
  - Input variables: acceptance status JSON, combined user data string  
  - System message: Detailed HTML email template for formatting  
- **OpenAI Chat Model1**  
  - Type: Language Model (OpenAI GPT-4o-mini)  
  - Role: Processes the email content prompt and returns generated email text.

---

#### 1.6 Storage of Enrollment Records  
**Overview:**  
Appends the enrollment submission data along with acceptance status into a Google Sheet for persistent record keeping.

**Nodes Involved:**  
- Append row in sheet

**Node Details:**  
- **Append row in sheet**  
  - Type: Google Sheets node  
  - Operation: Append new row with student data (name, email, WhatsApp, acceptance, programming basics, terms agreement, submission time)  
  - Config: Target spreadsheet ID and sheet name  
  - Credentials: Google Sheets OAuth2  
  - Potential failures: Permission errors, API quota exceedance, malformed data.

---

#### 1.7 Email Notification Dispatch  
**Overview:**  
Sends the acceptance or rejection email to the student's provided email address using Gmail OAuth2 authentication.

**Nodes Involved:**  
- Send a message in Gmail

**Node Details:**  
- **Send a message in Gmail**  
  - Type: Gmail node  
  - Operation: Sends email with subject and message generated by AI Agent1 output  
  - SendTo: Email from form submission  
  - Credentials: Gmail OAuth2  
  - Potential failures: OAuth token expiry, email quota limits, invalid recipient address.

---

### 3. Summary Table

| Node Name             | Node Type                          | Functional Role                        | Input Node(s)           | Output Node(s)          | Sticky Note                                                                                   |
|-----------------------|----------------------------------|-------------------------------------|------------------------|------------------------|-----------------------------------------------------------------------------------------------|
| On form submission     | Form Trigger                     | Captures enrollment form submission | -                      | Combine User Data       | This workflow starts **automatically** when a new student submits the enrollment form.        |
| Sticky Note            | Sticky Note                     | Documentation                       | -                      | -                      | This workflow starts **automatically** when a new student submits the enrollment form for the training program. |
| Combine User Data      | Set                             | Concatenates user info for AI input | On form submission     | Download file           |                                                                                               |
| Download file          | Google Drive                    | Downloads course enrollment spreadsheet | Combine User Data       | Pinecone Vector Store   |                                                                                               |
| Default Data Loader    | Document Loader                 | Loads spreadsheet as documents      | Download file           | Embeddings Google Gemini |                                                                                               |
| Embeddings Google Gemini | Embeddings Provider             | Creates text embeddings              | Default Data Loader     | Pinecone Vector Store   |                                                                                               |
| Pinecone Vector Store  | Vector Store (Pinecone)          | Inserts embeddings into vector store | Embeddings Google Gemini | AI Agent               |                                                                                               |
| AI Agent              | LangChain Agent                  | Evaluates acceptance eligibility    | Pinecone Vector Store   | AI Agent1               |                                                                                               |
| OpenAI Chat Model      | Language Model (OpenAI GPT-4o-mini) | Processes AI evaluation prompt     | AI Agent                | Structured Output Parser |                                                                                               |
| Structured Output Parser | Output Parser                    | Parses AI output JSON                | OpenAI Chat Model       | AI Agent1               |                                                                                               |
| AI Agent1             | LangChain Agent                  | Generates acceptance/rejection email | Structured Output Parser | Append row in sheet     |                                                                                               |
| OpenAI Chat Model1     | Language Model (OpenAI GPT-4o-mini) | Processes email content prompt     | AI Agent1               | Send a message in Gmail |                                                                                               |
| Append row in sheet    | Google Sheets                   | Logs enrollment data with acceptance | AI Agent1               | -                      |                                                                                               |
| Send a message in Gmail | Gmail                          | Sends enrollment status email       | OpenAI Chat Model1      | -                      |                                                                                               |
| Pinecone Vector Store1 | Vector Store (Pinecone)          | Retrieves data for email tool        | -                      | AI Agent1               |                                                                                               |
| Embeddings Google Gemini1 | Embeddings Provider             | Embeds data for Pinecone retrieval  | -                      | Pinecone Vector Store1  |                                                                                               |

---

### 4. Reproducing the Workflow from Scratch

1. **Create "On form submission" node**  
   - Type: Form Trigger  
   - Configure webhook to receive submissions  
   - Define form fields: Full Name (text, required), Email Address (email, required), WhatsApp Number (text, required), Programming Basics (dropdown: Yes/No, required), Commitment Checkbox (checkbox: Agree/Not Agree, limit selection 1, required), Terms and Conditions (dropdown: Agree/Don't agree, required)  
   - Add custom CSS for styling (as per original or simplified)  
   - Save and activate webhook trigger.

2. **Add "Combine User Data" node**  
   - Type: Set  
   - Create a single string field `user_context` concatenating form fields:  
     `{{ $json['Full Name'] }}, {{ $json['Email Address'] }}, {{ $json['WhatsApp Number'] }}, {{ $json['Do you have basics of any programming language?'] }}, {{ $json['Terms and Conditions'] }}`  
   - Connect output of "On form submission" to this node.

3. **Add "Download file" node**  
   - Type: Google Drive  
   - Operation: Download  
   - File ID: Google Sheet containing course enrollment data  
   - Google Drive OAuth2 Credentials configured  
   - Connect output of "Combine User Data" to this node.

4. **Add "Default Data Loader" node**  
   - Type: Document Default Data Loader  
   - Connect output of "Download file" node.

5. **Add "Embeddings Google Gemini" node**  
   - Type: Embeddings (Google Gemini)  
   - Configure Google Palm API credentials  
   - Connect output of "Default Data Loader" node.

6. **Add "Pinecone Vector Store" node**  
   - Type: Vector Store (Pinecone)  
   - Mode: Insert  
   - Pinecone namespace: "course_enrollment"  
   - Pinecone index: "course-enrollment"  
   - Configure Pinecone API credentials  
   - Connect output of "Embeddings Google Gemini" node.

7. **Add "AI Agent" node**  
   - Type: LangChain Agent  
   - Prompt: Define role as software testing instructor assistant, evaluate user answers to determine if accepted or not (accept if basics=yes, terms agreed, commitment agreed)  
   - Input: `user_context` from "Combine User Data"  
   - Connect "Pinecone Vector Store" main output to "AI Agent" input.  
   - Connect "OpenAI Chat Model" node (see next) as AI model.

8. **Add "OpenAI Chat Model" node**  
   - Type: LangChain OpenAI GPT-4o-mini  
   - Configure with OpenAI API credentials  
   - Connect as AI language model for "AI Agent".

9. **Add "Structured Output Parser" node**  
   - Type: Structured Output Parser  
   - Configure JSON schema example: `{ "accepted": true }`  
   - Connect "OpenAI Chat Model" output to this parser  
   - Connect parser output back to "AI Agent" node.

10. **Add "AI Agent1" node**  
    - Type: LangChain Agent  
    - Role: Mail agent to generate a professional acceptance or rejection email based on acceptance status and user data  
    - Use detailed HTML email template as system message  
    - Input: Output from previous AI Agent, combined user data  
    - Connect AI Agent output to "AI Agent1" input.

11. **Add "OpenAI Chat Model1" node**  
    - Type: LangChain OpenAI GPT-4o-mini  
    - API credentials: OpenAI  
    - Connect as AI language model for "AI Agent1".

12. **Add "Append row in sheet" node**  
    - Type: Google Sheets  
    - Operation: Append  
    - Configure target Spreadsheet ID and sheet name  
    - Map columns: Name, Email, WhatsApp, Accepted status, Programming basics, Terms agreement, Submission time from form submission and AI output  
    - Configure Google Sheets OAuth2 credentials  
    - Connect output of "AI Agent1" node.

13. **Add "Send a message in Gmail" node**  
    - Type: Gmail  
    - Operation: Send message  
    - Send to email from form submission  
    - Use message and subject generated by "OpenAI Chat Model1" (email content)  
    - Configure Gmail OAuth2 credentials  
    - Connect output of "OpenAI Chat Model1" node.

14. **Add auxiliary nodes for embedding and retrieval if needed:**  
    - "Pinecone Vector Store1" (retrieval mode) connected to "AI Agent1" for enhanced email personalization  
    - "Embeddings Google Gemini1" for embedding retrieval queries  

15. **Connect all nodes respecting the data flow as per the original workflow connection graph.**

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               | Context or Link                                                                                              |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------------------|
| The form includes detailed HTML content describing the course, schedule, payment plans, requirements, and mentorship links to ensure students are fully informed before submission.                                                                                                                                                                                                                                                                                                                                                                                                                                                         | The HTML form content embedded in the "On form submission" node.                                            |
| Rejected students are guided to watch the first 60 YouTube videos on programming basics for preparation, linking to a specific playlist: https://www.youtube.com/playlist?list=PL28DDB2DCF87BEE43                                                                                                                                                                                                                                                                                                                                                                                                    | Included in AI Agent1 email template to guide non-accepted students.                                        |
| Workflow uses multiple AI models and LangChain agents with OpenAI GPT-4o-mini and Google Gemini embeddings, requiring valid API credentials for Google Palm, OpenAI, and Pinecone.                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Credential configuration necessary for AI and vector store nodes.                                           |
| Custom form CSS applies a professional and modern style with responsive design and dark mode support, improving user experience.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       | CSS code embedded in "On form submission" node parameters.                                                  |
| The workflow logs all submissions and acceptance decisions in Google Sheets for audit and reporting.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | "Append row in sheet" node functionality.                                                                   |
| Pinecone is used to index enrollment data, enabling advanced semantic retrieval if workflow extended.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 | Potential extension for personalized AI interaction or historical data analysis.                            |
| Email notifications use a polished HTML template with conditional formatting for acceptance or rejection, ensuring clear communication.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                | System Message in AI Agent1 node.                                                                            |
| Workflow assumes stable internet connectivity and valid OAuth tokens for Google services (Drive, Sheets, Gmail). Tokens may require periodic refresh.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   | Operational consideration for credentials management.                                                       |

---

**Disclaimer:** The provided documentation exclusively describes an automated workflow created with n8n, adhering strictly to content policies and handling only legal, public data.