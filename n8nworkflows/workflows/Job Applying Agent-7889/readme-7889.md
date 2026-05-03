Job Applying Agent

https://n8nworkflows.xyz/workflows/job-applying-agent-7889


# Job Applying Agent

---

### 1. Workflow Overview

This workflow, titled **Job Applying Agent**, automates the process of submitting personalized job application emails to target businesses. It is designed for job seekers who want to streamline their applications by automatically extracting company information from provided business websites, retrieving related personal experience data, and generating customized emails for various job roles.

The workflow logically divides into these functional blocks:

- **1.1 Input Reception:** Captures user input from a form specifying the target business website and the job position applied for.
- **1.2 Website Data Retrieval and Processing:** Fetches the target business website content, converts it into a usable text format, and extracts key company information such as an email address and summary.
- **1.3 Validation and Conditional Routing:** Checks the extracted data for valid email presence to decide whether to proceed or stop.
- **1.4 Experience Data Retrieval:** Fetches user experience and portfolio details from a Google Sheet corresponding to the applied position.
- **1.5 AI-Driven Email Generation and Sending:** Uses AI language models to create a personalized job application email based on extracted company info and user experience, then sends it via Gmail.
- **1.6 User Feedback:** Provides the generated email response back to the user in the form interface.

---

### 2. Block-by-Block Analysis

#### 1.1 Input Reception

- **Overview:**  
Captures the target business URL and the job role to apply for via a web form trigger.

- **Nodes Involved:**  
  - On form submission

- **Node Details:**  
  - **On form submission**  
    - *Type & Role:* `Form Trigger` — starts workflow on submitted form data.  
    - *Configuration:* Form titled "Lead Agent" with two required fields: "Target Business Website" (text input) and "Applying As" (dropdown with options: Video Editor, SEO Expert, Full-Stack Developer, Social Media Manager).  
    - *Expressions/Variables:* Captures form data for downstream use.  
    - *Connections:* Outputs to the HTTP Request node for website content retrieval.  
    - *Edge Cases:* Missing or malformed URL; unsupported job role selection.  

---

#### 1.2 Website Data Retrieval and Processing

- **Overview:**  
Fetches raw HTML from the specified business website, converts HTML to markdown, and extracts key information (email, company summary) via AI.

- **Nodes Involved:**  
  - HTTP Request  
  - Markdown  
  - Information Extractor  
  - OpenAI Chat Model1

- **Node Details:**  
  - **HTTP Request**  
    - *Type & Role:* HTTP client — downloads the HTML content of the target URL from the form.  
    - *Configuration:* URL dynamically set from form input field "Target Business Website". No extra options configured.  
    - *Input:* From form submission node.  
    - *Output:* Raw HTML data.  
    - *Edge Cases:* Unreachable URL, HTTP errors, timeouts.  

  - **Markdown**  
    - *Type & Role:* Converts HTML content into markdown text for easier parsing.  
    - *Configuration:* Uses `$json.data` from HTTP Request node as input HTML.  
    - *Input:* HTTP Request output.  
    - *Output:* Markdown version of website content.  
    - *Edge Cases:* Incorrect HTML structure causing conversion issues.  

  - **OpenAI Chat Model1**  
    - *Type & Role:* AI language model (GPT-5) — likely processes or refines extracted content for Information Extractor.  
    - *Configuration:* Uses the "gpt-5" model; no additional options set.  
    - *Input:* Connected as AI language model resource for Information Extractor.  
    - *Output:* Processed text for information extraction.  
    - *Edge Cases:* API limits, invalid model name, network failures.  

  - **Information Extractor**  
    - *Type & Role:* AI-powered data extractor — extracts specific attributes: one email address and a company summary from the markdown text.  
    - *Configuration:* Input text is markdown output. Attributes specified: "Email Address" (required, single, must be valid email) and "Summary" (required, simple company/business summary).  
    - *Input:* Markdown output.  
    - *Output:* JSON object with extracted email and summary.  
    - *Edge Cases:* No email found, multiple conflicting emails, irrelevant summary, extraction failures.  

---

#### 1.3 Validation and Conditional Routing

- **Overview:**  
Validates if a valid email address was extracted; if yes, proceeds to AI email generation; otherwise, halts workflow.

- **Nodes Involved:**  
  - If  
  - No Operation, do nothing

- **Node Details:**  
  - **If**  
    - *Type & Role:* Conditional node — checks if extracted email contains '@' symbol (simple email validation).  
    - *Configuration:* Condition: `output["Email Address"]` contains '@'.  
    - *Input:* Information Extractor output.  
    - *Output:* True branch leads to AI Agent; False branch leads to No Operation node.  
    - *Edge Cases:* False negatives for valid emails missing '@'; false positives if email is malformed but contains '@'.  

  - **No Operation, do nothing**  
    - *Type & Role:* No-op — stops processing gracefully if no valid email found.  
    - *Configuration:* Default no parameters.  
    - *Input:* False branch from If node.  
    - *Output:* None.  

---

#### 1.4 Experience Data Retrieval

- **Overview:**  
Retrieves user’s experience and portfolio data for the applied position from a connected Google Sheet.

- **Nodes Involved:**  
  - Get row(s) in sheet in Google Sheets

- **Node Details:**  
  - **Get row(s) in sheet in Google Sheets**  
    - *Type & Role:* Data retrieval — accesses Google Sheet containing experience data.  
    - *Configuration:*  
      - Document ID: linked Google Sheet URL (contains skills, portfolio, and achievements).  
      - Sheet name: Sheet1 (gid=0).  
      - No filters specified (assumed to retrieve all data; likely filtered or processed later by AI Agent).  
    - *Input:* Connected as AI tool input to AI Agent node.  
    - *Output:* User experience data JSON.  
    - *Edge Cases:* Missing or inaccessible Google Sheet, permission errors, empty or malformed data.  

---

#### 1.5 AI-Driven Email Generation and Sending

- **Overview:**  
Generates a personalized job application email using AI based on the extracted company info and user experience, then sends it via Gmail.

- **Nodes Involved:**  
  - AI Agent  
  - OpenAI Chat Model  
  - Send a message in Gmail

- **Node Details:**  
  - **AI Agent**  
    - *Type & Role:* LangChain AI agent — orchestrates generation of the email content and subject using multiple inputs.  
    - *Configuration:*  
      - Input text composed dynamically using extracted company summary, email address, and applied position from form.  
      - System message instructs the AI to create a perfect email for the applied position, referencing experience data from Google Sheets.  
      - Prompt type: "define" (custom prompt with system message).  
    - *Input:* Receives company info (from Information Extractor), experience data (from Google Sheets), and job role (from form).  
    - *Output:* Generated email content, subject, and recipient email address.  
    - *Edge Cases:* AI API limits, prompt errors, mismatched or missing data inputs.  

  - **OpenAI Chat Model**  
    - *Type & Role:* GPT-5 language model — used by AI Agent as the language model backend for email generation.  
    - *Configuration:* Model set to "gpt-5".  
    - *Input:* Used internally by AI Agent node as AI language model resource.  
    - *Output:* Generated text response used by AI Agent.  
    - *Edge Cases:* API failures, model unavailability.  

  - **Send a message in Gmail**  
    - *Type & Role:* Email sender — sends the generated email via Gmail.  
    - *Configuration:*  
      - Recipient, subject, and message body dynamically set using AI Agent output overrides.  
      - Sender name set to "Jamal Mia" (user to update).  
      - Email type: plain text.  
      - Credentials: Gmail OAuth2 required.  
    - *Input:* AI Agent output (email fields).  
    - *Output:* Email sent confirmation.  
    - *Edge Cases:* Gmail API quota, authentication errors, invalid email addresses, network issues.  

---

#### 1.6 User Feedback

- **Overview:**  
Provides the AI-generated response back to the user through the form interface as a completion message.

- **Nodes Involved:**  
  - Form

- **Node Details:**  
  - **Form**  
    - *Type & Role:* Form node — sends back a completion message to the form submitter.  
    - *Configuration:*  
      - Operation: "completion".  
      - Completion title: "Agent Response".  
      - Completion message dynamically set to the AI Agent's output email content.  
    - *Input:* From AI Agent node completion output.  
    - *Output:* Provides feedback on the form UI.  
    - *Edge Cases:* UI rendering errors, missing AI output.  

---

### 3. Summary Table

| Node Name                  | Node Type                            | Functional Role                       | Input Node(s)             | Output Node(s)                 | Sticky Note                                                                                                           |
|----------------------------|------------------------------------|------------------------------------|---------------------------|-------------------------------|-----------------------------------------------------------------------------------------------------------------------|
| On form submission         | Form Trigger                       | Input reception (form data)         | —                         | HTTP Request                  |                                                                                                                       |
| HTTP Request               | HTTP Request                      | Fetch target business website HTML  | On form submission        | Markdown                      |                                                                                                                       |
| Markdown                   | Markdown                          | Convert HTML to markdown            | HTTP Request              | Information Extractor         |                                                                                                                       |
| OpenAI Chat Model1         | AI Language Model (OpenAI GPT-5) | Assist information extraction AI    | Information Extractor (AI) | Information Extractor         |                                                                                                                       |
| Information Extractor      | AI Information Extractor           | Extract email and company summary  | Markdown                  | If                           |                                                                                                                       |
| If                        | Conditional                      | Check valid email presence          | Information Extractor     | AI Agent, No Operation        |                                                                                                                       |
| No Operation, do nothing  | No Operation                      | Stop workflow if no valid email    | If (false branch)         | —                             |                                                                                                                       |
| Get row(s) in sheet in Google Sheets | Google Sheets Tool               | Retrieve user experience data       | AI Agent (ai_tool)        | AI Agent (ai_tool)             |                                                                                                                       |
| AI Agent                  | LangChain Agent                   | Generate personalized email content | If (true branch), Google Sheets | Form, Send a message in Gmail |                                                                                                                       |
| OpenAI Chat Model         | AI Language Model (OpenAI GPT-5) | Backend language model for AI Agent | AI Agent (ai_languageModel) | AI Agent                      |                                                                                                                       |
| Send a message in Gmail   | Gmail Tool                       | Send generated email                 | AI Agent (ai_tool)        | AI Agent                      |                                                                                                                       |
| Form                      | Form                             | Return AI response to user          | AI Agent                  | —                             |                                                                                                                       |
| Sticky Note               | Sticky Note                      | Workflow title                      | —                         | —                             | # Job Applier Agent - GPT 5                                                                                            |
| Sticky Note1              | Sticky Note                      | Setup guide and instructions        | —                         | —                             | ---\n# 🛠 Setup Guide\nFollow these steps to get started:\n1. **Set up the Job Applier Form**\nCustomize the form fields... |
| Sticky Note3              | Sticky Note                      | YouTube tutorial link                | —                         | —                             | ## Start here: Step by Step Youtube Tutorial :star:\n\n[![I Built an Auto Lead Finder AI Agent](https://img.youtube.com/vi/ysrBjL1aKkU/sddefault.jpg)](https://youtu.be/ysrBjL1aKkU) |

---

### 4. Reproducing the Workflow from Scratch

1. **Create Form Trigger node ("On form submission"):**  
   - Type: `Form Trigger`  
   - Configure form titled "Lead Agent" with two required fields:  
     - "Target Business Website" (text input)  
     - "Applying As" (dropdown with options: Video Editor, SEO Expert, Full-Stack Developer, Social Media Manager)  
   - Set response mode to "lastNode".  

2. **Create HTTP Request node:**  
   - Type: `HTTP Request`  
   - Connect input from "On form submission".  
   - URL parameter set dynamically: `={{ $json["Target Business Website"] }}`  
   - No special authentication or headers.  

3. **Create Markdown node:**  
   - Type: `Markdown`  
   - Input: connect from HTTP Request node.  
   - Parameter: convert the `$json.data` field (HTML) to markdown.  

4. **Create OpenAI Chat Model node (for info extraction support):**  
   - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
   - Set model to "gpt-5"  
   - Connect as AI language model resource to the "Information Extractor" node in the next step.  

5. **Create Information Extractor node:**  
   - Type: `@n8n/n8n-nodes-langchain.informationExtractor`  
   - Input text: `={{ $json.data }}` (markdown output)  
   - Attributes to extract:  
     - "Email Address" (required, single, validated as email format)  
     - "Summary" (required, a simple company/business summary)  
   - Connect input from Markdown node.  
   - Use OpenAI Chat Model node as AI language model resource.  

6. **Create If node:**  
   - Type: `If`  
   - Condition: Check if `output["Email Address"]` contains "@"  
   - Input from Information Extractor output.  
   - True branch connects to AI Agent node, False branch connects to No Operation node.  

7. **Create No Operation node ("No Operation, do nothing"):**  
   - Type: `No Operation`  
   - Input from If node false branch.  

8. **Create Google Sheets node ("Get row(s) in sheet in Google Sheets"):**  
   - Type: `Google Sheets`  
   - Connect as AI tool input to AI Agent node.  
   - Configure with your Google Sheet document ID containing experience data.  
   - Select "Sheet1" (gid=0).  
   - Ensure OAuth2 credentials for Google Sheets are configured.  

9. **Create AI Agent node:**  
   - Type: `@n8n/n8n-nodes-langchain.agent`  
   - Inputs:  
     - Company info and email from Information Extractor node  
     - Applied position from form submission  
     - Experience data from Google Sheets node  
   - Configuration:  
     - Text template to include:  
       ```
       Business/Company Info: {{ $('Information Extractor').item.json.output.Summary }}
       Email Address: {{ $('Information Extractor').item.json.output['Email Address'] }}

       Applied Position: {{ $('On form submission').item.json['Applying As'] }}
       ```  
     - System Message:  
       ```
       You task is to make a perfect Email based on the applied position for the Company/Business. And send the email with a perfect subject to the email address.
       My Name is Jamal Mia. And my working experience data is given for the applied position in the google Sheet.
       ```  
     - Prompt Type: "define"  
     - Connect AI language model input to the OpenAI Chat Model node (GPT-5).  
     - Connect AI tool input to Google Sheets node.  

10. **Create OpenAI Chat Model node (AI backend for AI Agent):**  
    - Type: `@n8n/n8n-nodes-langchain.lmChatOpenAi`  
    - Model: "gpt-5"  
    - Connect as AI language model resource to AI Agent node.  

11. **Create Gmail node ("Send a message in Gmail"):**  
    - Type: `Gmail`  
    - Connect AI tool input from AI Agent node (containing email fields).  
    - Parameters:  
      - Recipient: set dynamically from AI Agent output override "To".  
      - Message body: set dynamically from AI Agent output override "Message".  
      - Subject: set dynamically from AI Agent output override "Subject".  
      - Sender name: "Jamal Mia" (update to your name).  
      - Email type: plain text.  
    - OAuth2 credentials for Gmail required.  

12. **Create Form node (for user feedback):**  
    - Type: `Form`  
    - Connect input from AI Agent node (completion output).  
    - Operation: "completion"  
    - Completion title: "Agent Response"  
    - Completion message: set dynamically to AI Agent output field `output`.  

13. **Connect nodes according to the flow:**  
    - On form submission → HTTP Request → Markdown → Information Extractor → If  
    - If (true) → AI Agent → Form (completion)  
    - AI Agent → Gmail node  
    - If (false) → No Operation (stop)  
    - Google Sheets node connected as AI tool input to AI Agent  
    - OpenAI Chat Model nodes connected as AI language model inputs to Information Extractor and AI Agent respectively.  

14. **Credentials Setup:**  
    - Add OpenAI API credentials for both OpenAI Chat Model nodes.  
    - Add Google Sheets OAuth2 credentials for Google Sheets node.  
    - Add Gmail OAuth2 credentials for Gmail node.  

15. **Final verification:**  
    - Test form submission with valid URL and job role.  
    - Confirm website content fetch, data extraction, email generation, and sending works end-to-end.  
    - Adjust system messages and prompts as needed for personalization.  

---

### 5. General Notes & Resources

| Note Content                                                                                                                                                                                                                                                                             | Context or Link                                                                                                                         |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------|
| Setup guide included as a sticky note in the workflow: stepwise instructions for form setup, API credentials, Google Sheets linking, Gmail connection, and AI prompt customization.                                                                                                        | See Sticky Note1 in workflow for full setup instructions.                                                                               |
| YouTube tutorial video titled "I Built an Auto Lead Finder AI Agent" provides a visual walkthrough for similar workflow concepts and setup.                                                                                                                                            | https://youtu.be/ysrBjL1aKkU (linked in Sticky Note3 with embedded thumbnail).                                                         |
| User must update Gmail sender name and Google Sheets experience data to reflect their actual profile and skills.                                                                                                                                                                         | Gmail node and Google Sheets node configuration.                                                                                        |
| The workflow is designed around GPT-5 model usage; ensure you have access to this model or adjust to available OpenAI models accordingly.                                                                                                                                              | OpenAI API subscription and model availability required.                                                                                |
| Email address extraction uses simple validation (presence of '@'); consider extending validation for better accuracy if needed.                                                                                                                                                        | If node condition.                                                                                                                      |

---

disclaimer Le texte fourni provient exclusivement d’un workflow automatisé réalisé avec n8n, un outil d’intégration et d’automatisation. Ce traitement respecte strictement les politiques de contenu en vigueur et ne contient aucun élément illégal, offensant ou protégé. Toutes les données manipulées sont légales et publiques.

---